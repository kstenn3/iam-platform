# Week 1, Step 2 — Database Layer

A walkthrough of what each piece does and why, rather than code to paste. Work top to bottom; each unit builds on the last.

---

## 2a — Docker Compose

### The problem

You need PostgreSQL locally. Installing it on Windows directly works, but then your machine runs one specific version configured one specific way, forever. When you deploy to RDS on a different version, you discover the difference through bugs.

A container gives you an exact pinned version, defined in a file you commit. Anyone who clones the repo gets byte-identical infrastructure with one command.

Second payoff: containers are a documented resume gap. This file starts closing it.

### `docker-compose.yml` — repo root

```yaml
services:
  postgres:
    image: postgres:17-alpine
    container_name: iam-postgres
    environment:
      POSTGRES_USER: iam
      POSTGRES_PASSWORD: localdev
      POSTGRES_DB: iamplatform
    ports:
      - "5432:5432"
    volumes:
      - iam-pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U iam"]
      interval: 5s
      retries: 5

volumes:
  iam-pgdata:
```

Redis is deliberately absent. It's not needed until week 2, and config you can't explain is worse than config you don't have.

### What each line does

**`image: postgres:17-alpine`** — `17` pins the major version. Without a tag you get `latest`, which changes underneath you. `alpine` is a minimal base, ~80MB versus ~400MB. Same discipline as pinning NuGet versions.

**`container_name`** — stable name so `docker exec -it iam-postgres` works instead of a generated one.

**`environment`** — the Postgres image reads these on **first startup against an empty volume** to create the initial user and database. Changing the password later does nothing, because initialization already ran. This confuses everyone once.

**`ports: "5432:5432"`** — host port left, container port right. Containers are network-isolated; this maps a host port through. If something already held 5432 you'd write `"5433:5432"`.

**`volumes: iam-pgdata:/var/lib/postgresql/data`** — the important one. **Container filesystems are ephemeral** — delete the container and its contents die. A named volume is storage that outlives the container, mounted where Postgres writes. Without this, `docker compose down` wipes your database.

**`healthcheck`** — runs `pg_isready` every 5s to distinguish "process started" from "accepting connections." Postgres takes seconds to initialize; that gap matters when other services depend on it.

### Run and verify

```powershell
docker compose up -d
docker ps
docker exec -it iam-postgres psql -U iam -d iamplatform -c "\l"
```

`-d` detaches. `docker exec` runs a command inside a running container; `-it` gives it a terminal.

**Answers to the two questions:**

1. Data survives `down` then `up` — the named volume is a separate object from the container. Only `docker compose down -v` destroys it.
2. `localhost:5432` works because the port mapping forwards the host's 5432 into the container. Your app never knows a container is involved.

---

## 2b — The `ApplicationUser` entity

### The design decision behind it

You're using **ASP.NET Core Identity for authentication** and **your own model for authorization**. Worth being able to defend:

Identity handles credentials — PBKDF2 password hashing, lockout, security stamps. Rolling your own password storage is a genuine career risk; Identity is the correct engineering answer.

But Identity's roles are flat strings. You need hierarchy, group-derived roles, and resource scoping. `[Authorize(Roles = "Admin")]` cannot express "admin of org A, viewer of org B." So `AspNetRoles` goes unused and you build `Roles`, `Permissions`, and `RoleAssignments` yourself in week 2.

That's the answer when an interviewer asks why you didn't just use Identity's roles.

### `src/IamPlatform.Domain/Entities/ApplicationUser.cs`

```csharp
using Microsoft.AspNetCore.Identity;

namespace IamPlatform.Domain.Entities;

public class ApplicationUser : IdentityUser<Guid>
{
    public string? DisplayName { get; set; }

    public bool IsActive { get; set; } = true;
    public DateTimeOffset? DeactivatedAt { get; set; }

    public int PermissionEpoch { get; set; } = 1;

    public DateTimeOffset CreatedAt { get; set; } = DateTimeOffset.UtcNow;
}
```

### Why each piece

**`IdentityUser<Guid>`** — inheriting brings `Id`, `UserName`, `Email`, `PasswordHash`, `SecurityStamp`, `LockoutEnd`, and more. The generic parameter sets the key type.

**Why `Guid` over the default `string`** — Identity defaults to string keys holding a GUID's text form. Storing them as native `uuid` in Postgres is smaller, indexes better, and avoids string comparison on every join. Sequential integer keys would be worse here: they leak how many users exist and let someone enumerate `/users/1`, `/users/2`.

**`IsActive` + `DeactivatedAt` — soft delete.** Hard-deleting a user orphans every audit record referencing them. You lose who granted a permission and who acted on it, which is precisely what audit exists to answer. Deactivated users keep their identity and history but can't authenticate. Real IAM systems all work this way, and knowing why is the point.

**`PermissionEpoch`** — the architectural centerpiece, present in week 1 so you don't need a second migration.

The problem: JWTs are stateless. That's why they scale — no database round trip to validate. It's also why revoking a role doesn't affect tokens already issued; they carry the old claims until expiry.

The mechanism: embed the user's current epoch as a claim at token issue time. On each request, compare the token's epoch against the cached current value. Any role, group, or activation change bumps the epoch, and every outstanding token for that user fails the comparison instantly.

Cost: one integer comparison against a cached value. Compare that to introspecting every token against the database, which works but discards the entire reason to use JWTs.

**`DateTimeOffset` rather than `DateTime`** — carries UTC offset explicitly. `DateTime` has a `Kind` that gets lost across serialization boundaries and causes timezone bugs that surface in production.

---

## 2c — The DbContext

### What EF Core is doing

A `DbContext` is two things at once: a map from C# classes to database tables, and a unit of work tracking changes so `SaveChanges()` can write them in one transaction.

`IdentityDbContext` is a `DbContext` that already knows about Identity's tables, so you inherit rather than declaring seven `DbSet` properties by hand.

### `src/IamPlatform.Infrastructure/IamDbContext.cs`

```csharp
using IamPlatform.Domain.Entities;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace IamPlatform.Infrastructure;

public class IamDbContext : IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>
{
    public IamDbContext(DbContextOptions<IamDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        builder.Entity<ApplicationUser>(e =>
        {
            e.Property(u => u.DisplayName).HasMaxLength(200);
            e.HasIndex(u => u.IsActive);
        });

        builder.UseOpenIddict();
    }
}
```

### Why each piece

**`IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>`** — three parameters: your user type, the role type, the key type. All three must agree on `Guid` or you get compile errors that read as nonsense.

**The constructor taking `DbContextOptions`** — the connection string and provider get injected rather than hardcoded. That's what lets tests swap Postgres for an in-memory provider without touching this class.

**`base.OnModelCreating(builder)` first** — Identity configures its own tables in the base call. Omit it and nothing works. Your customizations come after.

**`HasMaxLength(200)`** — without it EF generates `text` with no bound. Constraints at the database level catch bugs the application layer misses.

**`HasIndex(u => u.IsActive)`** — you'll filter by this constantly ("show active users"). An index turns a table scan into a lookup. Indexes are a tradeoff: faster reads, slower writes, more disk. Worth it on a column in most queries.

**`builder.UseOpenIddict()`** — adds OpenIddict's four tables (applications, authorizations, scopes, tokens) to this context. Registered clients, issued authorization codes, and refresh tokens all live there.

---

## 2d — Program.cs and dependency injection

### What DI is doing

ASP.NET Core builds a container at startup mapping interfaces to implementations. When a controller needs a `UserManager<ApplicationUser>`, it declares it as a constructor parameter and the framework supplies one. Nothing calls `new`.

The payoff is testability — swap real implementations for fakes without changing the class under test — and lifetime management, which the container handles.

### `src/IamPlatform.Server/Program.cs`

```csharp
using IamPlatform.Domain.Entities;
using IamPlatform.Infrastructure;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<IamDbContext>(options =>
{
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default"));
    options.UseOpenIddict();
});

builder.Services
    .AddIdentity<ApplicationUser, IdentityRole<Guid>>(options =>
    {
        options.Password.RequiredLength = 12;
        options.User.RequireUniqueEmail = true;
        options.Lockout.MaxFailedAccessAttempts = 5;
    })
    .AddEntityFrameworkStores<IamDbContext>()
    .AddDefaultTokenProviders();

var app = builder.Build();
app.MapGet("/", () => "IAM Platform");
app.Run();
```

### Why each piece

**`AddDbContext<IamDbContext>`** — registers the context as **scoped**, meaning one instance per HTTP request. This matters: `DbContext` is not thread-safe and accumulates change-tracking state. A singleton would leak memory and corrupt data across requests; a transient would break unit-of-work semantics within one request.

**`UseNpgsql(...)`** — the Postgres provider. EF Core is provider-agnostic; this is the line that decides which dialect gets generated.

**`options.UseOpenIddict()` inside `AddDbContext`** — distinct from the call in `OnModelCreating`. That one shaped the schema; this one registers OpenIddict's EF stores.

**Password `RequiredLength = 12`** — Identity defaults to 6, which is too short. Length beats complexity rules; NIST guidance moved away from mandatory symbols and rotation years ago, since both push users toward predictable patterns. A defensible choice you can explain beats an arbitrary one.

**`Lockout.MaxFailedAccessAttempts = 5`** — brute-force protection. On an IAM project, being asked "how do you prevent credential stuffing?" is likely, and lockout is half the answer. Rate limiting is the other half, and it's on the cut list.

**`AddEntityFrameworkStores<IamDbContext>()`** — tells Identity to persist through your context rather than some other store.

**`AddDefaultTokenProviders()`** — generates tokens for password reset and email confirmation. Both are cut from v1, but the providers cost nothing and their absence produces confusing runtime errors later.

### A gotcha you will hit

`AddIdentity` sets the default authentication scheme to cookies. OpenIddict wants to control authentication on its own endpoints. When you wire up the authorization endpoint in step 3, expect scheme conflicts — the usual resolution is being explicit about which scheme applies where, or using `AddIdentityCore` and adding the cookie handler yourself.

Flagging it now so it reads as expected rather than broken.

---

## 2e — user-secrets

### The problem

Your connection string contains a password. `appsettings.Development.json` is committed. Committing credentials in an IAM repo is the single most embarrassing thing a reviewer could find, and reviewers do look.

### How configuration layering works

ASP.NET Core reads config sources in order, later overriding earlier:

1. `appsettings.json`
2. `appsettings.{Environment}.json`
3. **User secrets** (Development only)
4. Environment variables
5. Command-line arguments

Same key, highest-priority source wins. So local development reads from user secrets, and AWS reads the same key from an environment variable. No code change between them.

### Commands

```powershell
dotnet user-secrets init --project src/IamPlatform.Server
dotnet user-secrets set "ConnectionStrings:Default" "Host=localhost;Port=5432;Database=iamplatform;Username=iam;Password=localdev" --project src/IamPlatform.Server
```

`init` adds a `UserSecretsId` GUID to the `.csproj`. That GUID is committed and is not a secret — it's just a folder name.

The values land in `%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json`, outside the repo entirely. Not encrypted — protection comes from location, not cryptography. Adequate for local development, not for production, which is what environment variables and AWS Secrets Manager are for.

The `:` in `ConnectionStrings:Default` is section nesting — equivalent to `{ "ConnectionStrings": { "Default": "..." } }`.

---

## 2f — Migrations

### What a migration is

EF Core compares your entity classes against a snapshot of the last known schema and generates C# describing the difference — `CreateTable`, `AddColumn`, `CreateIndex`. Each migration has an `Up` (apply) and a `Down` (reverse).

They're code, so they're versioned, reviewable, and run identically on your laptop and in production. The alternative — hand-written SQL scripts applied by hand — is how schemas drift between environments.

### Commands

```powershell
dotnet tool install --global dotnet-ef
dotnet ef migrations add InitialIdentity -p src/IamPlatform.Infrastructure -s src/IamPlatform.Server
dotnet ef database update -p src/IamPlatform.Infrastructure -s src/IamPlatform.Server
```

### The `-p` versus `-s` distinction

This trips up nearly everyone.

- **`-p` (project)** — where the migration files get written and where `DbContext` lives: `Infrastructure`
- **`-s` (startup project)** — where the app's configuration and DI live, so the tool can build a context with a real connection string: `Server`

Two projects because `Infrastructure` holds the context but has no configuration; `Server` holds configuration but not the context. The tool needs both.

### What to inspect before running `update`

Open the generated file in `src/IamPlatform.Infrastructure/Migrations/`. Read the `Up` method. You should recognize:

- `AspNetUsers` with your `DisplayName`, `IsActive`, `PermissionEpoch`, `CreatedAt` columns
- `AspNetRoles`, `AspNetUserRoles`, `AspNetUserClaims`, and the rest of Identity's tables
- Four `OpenIddict*` tables
- An index on `IsActive`

**Read migrations before applying them, every time.** A misconfigured mapping can generate a `DropTable` you didn't intend. In production that's a catastrophe. Building the habit now on a project where mistakes are free is the point.

### Verify

```powershell
docker exec -it iam-postgres psql -U iam -d iamplatform -c "\dt"
docker exec -it iam-postgres psql -U iam -d iamplatform -c "\d ""AspNetUsers"""
```

The second command describes the table — confirm your four custom columns are present with the types you expect.

Also note `__EFMigrationsHistory`. That's how EF tracks which migrations have been applied, and it's why running `update` twice is safe.

---

## Checkpoint

Before step 3, you should be able to answer:

1. Why does the named volume exist in `docker-compose.yml`?
2. Why soft-delete users instead of removing the row?
3. What problem does `PermissionEpoch` solve, and what's the alternative you rejected?
4. Why is `DbContext` registered as scoped rather than singleton?
5. What's the difference between `-p` and `-s` in the EF commands?

These aren't quiz questions for their own sake — 3 and 4 in particular are things you'll be asked in a technical deep dive on this project.

Next: OpenIddict configuration and the authorization code flow with PKCE.
