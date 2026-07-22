---
name: abp-dependency-rules
description: ABP project layer dependency rules - which projects can reference which, domain/application/infrastructure separation, cross-layer violations to avoid. Use when reviewing project structure, adding new project references, or checking if a dependency direction is correct.
user-invocable: false
---

# ABP Dependency Rules

ABP enforces strict layer separation to maintain clean architecture. Violating these rules leads to tight coupling, untestable code, and broken modularity.

## Core Principles (All Templates)

These principles apply regardless of solution structure:

1. **Domain logic never depends on infrastructure** — No DbContext in domain/application layers
2. **Use abstractions** (interfaces) for dependencies — Depend on contracts, not implementations
3. **Higher layers depend on lower layers**, never the reverse
4. **Data access through repositories**, not direct DbContext

## Layered Template Structure

> This section applies to layered templates (app, module). Single-layer and microservice templates have different structures.

### Layer Diagram

```
┌─────────────────────────────────────────────┐
│  Host (Web / API)                           │
│  - DI wiring, middleware, startup            │
│  - References: HttpApi, Application         │
└─────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────┐
│  HttpApi                                    │
│  - REST controllers                         │
│  - References: Application.Contracts ONLY   │
└─────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────┐
│  Application                                │
│  - App service implementations              │
│  - References: Domain, Application.Contracts│
└─────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────┐
│  Application.Contracts                      │
│  - App service interfaces, DTOs             │
│  - References: Domain.Shared ONLY           │
└─────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────┐
│  Domain                                     │
│  - Entities, repository interfaces,         │
│    domain services                          │
│  - References: Domain.Shared ONLY           │
└─────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────┐
│  Domain.Shared                              │
│  - Constants, enums, localization keys      │
│  - References: NOTHING                      │
└─────────────────────────────────────────────┘
```

### Data Access Layers (EF Core / MongoDB)

```
┌─────────────────────────────────────────────┐
│  EntityFrameworkCore / MongoDB              │
│  - DbContext, repository implementations    │
│  - References: Domain ONLY                  │
└─────────────────────────────────────────────┘
```

### Dependency Direction Table

| Project | Can Reference | Referenced By |
|---------|---------------|---------------|
| `Domain.Shared` | Nothing | All projects |
| `Domain` | Domain.Shared | Application, EF Core, MongoDB |
| `Application.Contracts` | Domain.Shared | Application, HttpApi, HttpApi.Client |
| `Application` | Domain, Application.Contracts | Host |
| `EntityFrameworkCore` | Domain only | Host |
| `MongoDB` | Domain only | Host |
| `HttpApi` | Application.Contracts only | Host |
| `HttpApi.Client` | Application.Contracts only | Clients, Gateways |

## Critical Rules

### ❌ Never Do

```csharp
// 1. Application layer accessing DbContext directly
public class BookAppService : ApplicationService
{
    private readonly MyDbContext _dbContext; // ❌ WRONG — use repository
}

// 2. Domain depending on application layer
public class BookManager : DomainService
{
    private readonly IBookAppService _appService; // ❌ WRONG — circular dependency
}

// 3. HttpApi depending on Application implementation
public class BookController : AbpController
{
    private readonly BookAppService _bookAppService; // ❌ WRONG — use interface
}

// 4. Domain referencing EF Core / MongoDB
// Domain project MUST NOT have package reference to:
//   - Volo.Abp.EntityFrameworkCore
//   - Volo.Abp.MongoDB
//   - Microsoft.EntityFrameworkCore

// 5. Application.Contracts referencing Domain
// Application.Contracts MUST NOT reference Domain project
// It can only reference Domain.Shared
```

### ✅ Always Do

```csharp
// 1. Application layer using repository abstraction
public class BookAppService : ApplicationService
{
    private readonly IBookRepository _bookRepository; // ✅ CORRECT
}

// 2. Domain service using domain abstractions
public class BookManager : DomainService
{
    private readonly IBookRepository _bookRepository; // ✅ CORRECT
}

// 3. HttpApi depending on contracts only
public class BookController : AbpController
{
    private readonly IBookAppService _bookAppService; // ✅ CORRECT — interface from Contracts
}

// 4. Domain.Shared has zero dependencies
// ✅ CORRECT — no project references at all
```

## Repository Pattern Enforcement

### Interface Location (Domain Layer)

```csharp
// In Domain project — only the abstraction
public interface IBookRepository : IRepository<Book, Guid>
{
    Task<Book> FindByNameAsync(string name);
}
```

### Implementation Location (Data Layer)

```csharp
// In EntityFrameworkCore project — the EF Core implementation
public class BookRepository : EfCoreRepository<MyDbContext, Book, Guid>, IBookRepository
{
    public BookRepository(IDbContextProvider<MyDbContext> dbContextProvider)
        : base(dbContextProvider) { }

    public async Task<Book> FindByNameAsync(string name)
    {
        var dbContext = await GetDbContextAsync();
        return await dbContext.Books.FirstOrDefaultAsync(b => b.Name == name);
    }
}

// In MongoDB project — the MongoDB implementation
public class BookRepository : MongoDbRepository<MyMongoDbContext, Book, Guid>, IBookRepository
{
    public BookRepository(IMongoDbContextProvider<MyMongoDbContext> dbContextProvider)
        : base(dbContextProvider) { }

    public async Task<Book> FindByNameAsync(string name)
    {
        var queryable = await GetMongoQueryableAsync();
        return await queryable.FirstOrDefaultAsync(b => b.Name == name);
    }
}
```

## Multi-Application Scenarios

When you have multiple applications (e.g., Admin + Public API):

### Vertical Separation

```
MyProject.Admin.Application      — Admin-specific services
MyProject.Public.Application     — Public-specific services
MyProject.Domain                 — Shared domain (both reference this)
MyProject.Domain.Shared          — Shared constants/enums (all reference this)
```

### Rules for Multi-Application

- Admin and Public application layers **MUST NOT** reference each other
- Share domain logic, not application logic
- Each vertical can have its own DTOs even if similar
- Each vertical has its own `Application.Contracts`

## Common Violations

| Violation | Why It Happens | How to Fix |
|-----------|---------------|------------|
| Injecting `DbContext` in Application | Developer wants direct query access | Create a repository method |
| Domain referencing Application | Circular logic need | Extract to Domain Service |
| HttpApi referencing Application | Convenience | Reference only `Application.Contracts` |
| `Application.Contracts` referencing `Domain` | Need entity types in DTOs | Use DTOs, not entities; reference `Domain.Shared` only |
| Business logic in Controller | Quick implementation | Move to Application Service |
| Direct SQL in Application Service | Performance concern | Create repository method with raw SQL |

## Enforcement Checklist

- `Domain.Shared` has zero project references
- `Domain` only references `Domain.Shared`
- `Application.Contracts` only references `Domain.Shared`
- `Application` only references `Domain` and `Application.Contracts`
- `EntityFrameworkCore` only references `Domain`
- `MongoDB` only references `Domain`
- `HttpApi` only references `Application.Contracts`
- `HttpApi.Client` only references `Application.Contracts`
- No `DbContext` injected in Application layer
- No domain logic in Controllers
- Repository interfaces in Domain, implementations in data layer
- No cross-references between sibling application modules

## Enforcement Checklist (Layered Templates)

When adding a new feature:
1. **Entity changes?** → Domain project
2. **Constants/enums?** → Domain.Shared project
3. **Repository interface?** → Domain project (only if custom queries needed)
4. **Repository implementation?** → EntityFrameworkCore/MongoDB project
5. **DTOs and service interface?** → Application.Contracts project
6. **Service implementation?** → Application project
7. **API endpoint?** → HttpApi project (if not using auto API controllers)

## Common Violations to Watch

| Violation | Impact | Fix |
|-----------|--------|-----|
| DbContext in Application | Breaks DB independence | Use repository |
| Entity in DTO | Exposes internals | Map to DTO |
| IQueryable in interface | Breaks abstraction | Return concrete types |
| Cross-module app service call | Tight coupling | Use events or domain |
