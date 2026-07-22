---
name: abp-core
description: ABP Framework fundamentals - module system, DI registration, base classes (ApplicationService, DomainService), IClock, BusinessException, localization, async patterns. Use when configuring modules, registering services, handling time/exceptions, or needing foundational ABP conventions.
user-invocable: false
---

# ABP Core Conventions

ABP Framework provides a modular, layered architecture with built-in dependency injection, localization, exception handling, and async patterns. These conventions apply to ALL ABP projects.

## Key Rules

- Use `IClock` / `Clock.Now` instead of `DateTime.Now` / `DateTime.UtcNow`
- Use `ITransientDependency` / `IScopedDependency` / `ISingletonDependency` marker interfaces instead of manual `AddScoped`/`AddTransient`/`AddSingleton`
- Use `IRepository<T>` instead of injecting `DbContext` directly
- Check base class properties (`Clock`, `CurrentUser`, `GuidGenerator`, `L`) before injecting services
- Use `BusinessException` with namespaced error codes for domain rule violations
- All async methods must end with `Async` suffix

## Module System

Every ABP application/module has a module class that configures services. Modules declare dependencies via `[DependsOn]`:

```csharp
[DependsOn(
    typeof(AbpDddDomainModule),
    typeof(AbpEntityFrameworkCoreModule)
)]
public class MyAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // Service registration and configuration
        context.Services.AddAbpDbContext<MyAppDbContext>(options =>
        {
            options.AddDefaultRepositories();
        });
    }

    public override void OnApplicationInitialization(ApplicationInitializationContext context)
    {
        // Middleware configuration — only in final host application
        var app = context.GetApplicationBuilder();
        app.UseRouting();
        app.UseMvc();
    }
}
```

> Middleware configuration (`OnApplicationInitialization`) should only be done in the **final host application**, not in reusable modules.

## Dependency Injection Conventions

### Automatic Registration via Marker Interfaces

ABP automatically registers services implementing these marker interfaces:

| Interface | Lifetime | Use When |
|-----------|----------|----------|
| `ITransientDependency` | Transient | Lightweight, stateless services |
| `IScopedDependency` | Scoped | Per-request services |
| `ISingletonDependency` | Singleton | Application-wide state |

```csharp
public class MyService : ITransientDependency
{
    // Automatically registered as Transient
}

public class MyCacheService : ISingletonDependency
{
    // Automatically registered as Singleton
}
```

### Auto-Registered Base Classes

Classes inheriting from these base classes are also auto-registered:

| Base Class | Lifetime |
|-----------|----------|
| `ApplicationService` | Transient |
| `DomainService` | Transient |
| `AbpController` | Transient |
| `AbpPageModel` | Transient |

### Exposing Services

Use `[ExposeServices]` to control which interfaces a service is exposed as:

```csharp
[ExposeServices(typeof(IMyService), typeof(IMyOtherService))]
public class MyService : IMyService, IMyOtherService, ITransientDependency
{
}
```

### Repository Usage

Use the generic `IRepository<TEntity, TKey>` for simple CRUD. Define custom repository interfaces only when you need custom query methods:

```csharp
// Simple CRUD — Generic repository is fine
public class BookAppService : ApplicationService
{
    private readonly IRepository<Book, Guid> _bookRepository; // ✅ OK for simple operations

    public async Task<BookDto> GetAsync(Guid id)
    {
        var book = await _bookRepository.GetAsync(id);
        return ObjectMapper.Map<Book, BookDto>(book);
    }
}

// Custom queries needed — Define custom interface
public interface IBookRepository : IRepository<Book, Guid>
{
    Task<Book> FindByNameAsync(string name);
}

public class BookAppService : ApplicationService
{
    private readonly IBookRepository _bookRepository; // ✅ Use custom when needed
}
```

## Important Base Classes

| Base Class | Purpose | Layer |
|------------|---------|-------|
| `Entity<TKey>` | Basic entity with ID | Domain |
| `AggregateRoot<TKey>` | DDD aggregate root | Domain |
| `DomainService` | Domain business logic | Domain |
| `ApplicationService` | Use case orchestration | Application |
| `AbpController` | REST API controller | HttpApi |
| `AbpPageModel` | Razor Page model | Web |

### Base Class Pre-Injected Properties

ABP base classes already inject commonly used services as properties. **Before injecting a service, check if it's already available:**

| Property | Available In | Type | Description |
|----------|-------------|------|-------------|
| `GuidGenerator` | All base classes | `IGuidGenerator` | Generate sequential GUIDs |
| `Clock` | All base classes | `IClock` | Current time (use instead of `DateTime`) |
| `CurrentUser` | All base classes | `ICurrentUser` | Authenticated user info |
| `CurrentTenant` | All base classes | `ICurrentTenant` | Multi-tenancy context |
| `L` | `ApplicationService`, `AbpController`, `AbpPageModel` | `IStringLocalizer` | Localization |
| `AuthorizationService` | `ApplicationService`, `AbpController` | `IAuthorizationService` | Permission checks |
| `FeatureChecker` | `ApplicationService`, `AbpController` | `IFeatureChecker` | Feature availability |
| `DataFilter` | All base classes | `IDataFilter` | Soft-delete/tenant filter control |
| `UnitOfWorkManager` | `ApplicationService`, `DomainService` | `IUnitOfWorkManager` | Unit of work management |
| `LoggerFactory` | All base classes | `ILoggerFactory` | Create loggers |
| `Logger` | All base classes | `ILogger` | Auto-created logger |
| `ObjectMapper` | `ApplicationService` | `IObjectMapper` | Object-to-object mapping |
| `LazyServiceProvider` | All base classes | `ILazyServiceProvider` | Lazy service resolution |

### Useful Methods from Base Classes

```csharp
// In ApplicationService or AbpController:
await CheckPolicyAsync("MyPermission");  // Check permission, throw if not granted
bool granted = await IsGrantedAsync("MyPermission");  // Check without throwing
```

## Async Best Practices

- Use async all the way — never use `.Result` or `.Wait()`
- All async methods should end with `Async` suffix
- ABP automatically handles `CancellationToken` in most cases (from `HttpContext.RequestAborted`)
- Only pass `CancellationToken` explicitly when implementing custom cancellation logic
- Never use `async void` — always return `Task` or `Task<T>`

```csharp
// ✅ Correct
public async Task<BookDto> GetAsync(Guid id)
{
    var book = await _bookRepository.GetAsync(id);
    return ObjectMapper.Map<Book, BookDto>(book);
}

// ❌ Wrong
public BookDto Get(Guid id)
{
    var book = _bookRepository.GetAsync(id).Result; // Deadlock risk!
    return ObjectMapper.Map<Book, BookDto>(book);
}
```

## Time Handling

Never use `DateTime.Now` or `DateTime.UtcNow` directly. Use ABP's `IClock` service:

```csharp
// In classes inheriting from base classes (ApplicationService, DomainService, etc.)
public class BookAppService : ApplicationService
{
    public void DoSomething()
    {
        var now = Clock.Now; // ✅ Already available as property
    }
}

// In other services — inject IClock
public class MyService : ITransientDependency
{
    private readonly IClock _clock;

    public MyService(IClock clock) => _clock = clock;

    public void DoSomething()
    {
        var now = _clock.Now; // ✅ Correct — testable, respects timezone settings
        // var now = DateTime.Now; // ❌ Wrong — not testable, ignores timezone
    }
}
```

## Business Exceptions

Use `BusinessException` for domain rule violations with namespaced error codes:

```csharp
throw new BusinessException("MyModule:BookNameAlreadyExists")
    .WithData("Name", bookName);
```

Configure localization mapping in module:

```csharp
Configure<AbpExceptionLocalizationOptions>(options =>
{
    options.MapCodeNamespace("MyModule", typeof(MyModuleResource));
});
```

Error code convention: `ModuleName:ErrorCode` (e.g., `Identity:UserNameAlreadyExists`, `BookStore:BookNameAlreadyExists`).

## Localization

### In Base Classes

Use `L["Key"]` — the `IStringLocalizer` property available in `ApplicationService`, `AbpController`, `AbpPageModel`:

```csharp
public class BookAppService : ApplicationService
{
    public void Notify()
    {
        var message = L["BookCreated"]; // ✅ Localized string
    }
}
```

### In Other Services

Inject `IStringLocalizer<TResource>`:

```csharp
public class MyService : ITransientDependency
{
    private readonly IStringLocalizer<BookStoreResource> _localizer;

    public MyService(IStringLocalizer<BookStoreResource> localizer)
    {
        _localizer = localizer;
    }
}
```

### Localization File Location

`*.Domain.Shared/Localization/{ResourceName}/{lang}.json`

```json
// Example: MyProject.Domain.Shared/Localization/BookStore/en.json
{
  "culture": "en",
  "texts": {
    "Menu:Home": "Home",
    "Welcome": "Welcome",
    "BookName": "Book Name",
    "BookCreated": "Book created successfully"
  }
}
```

### Localization Resource Definition

```csharp
public class BookStoreResource
{
}
```

Configure in module:

```csharp
Configure<AbpLocalizationOptions>(options =>
{
    options.Resources
        .Add<BookStoreResource>("en")
        .AddVirtualJson("/Localization/BookStore");
});
```

## Anti-Patterns

| Don't Use | Use Instead | Why |
|-----------|-------------|-----|
| `DateTime.Now` / `DateTime.UtcNow` | `IClock.Now` / `Clock.Now` | Testable, respects timezone settings |
| `AddScoped<T>()` / `AddTransient<T>()` | `ITransientDependency` / `IScopedDependency` | Convention-based, cleaner |
| `DbContext` injection in Application | `IRepository<T>` or custom repository | Proper layering, testability |
| `.Result` / `.Wait()` on async calls | `await` all the way | Avoids deadlocks |
| `async void` | `async Task` | Exception handling, caller awareness |
| Hardcoded strings for user messages | `L["Key"]` / `IStringLocalizer<T>` | Localization support |
| Generic error messages | `BusinessException` with error codes | Proper error handling, localizable |
| Minimal APIs (for ABP apps) | ABP Controllers (`AbpController`) | Auto API, authorization, validation integration |
| MediatR (for ABP apps) | Application Services | Built-in patterns, no extra library |

## Best Practices Checklist

- Use `IClock` instead of `DateTime`
- Use marker interfaces for DI registration
- Use `IRepository<T>` instead of `DbContext` in Application layer
- Check base class properties before injecting services
- All async methods end with `Async` suffix
- Use `BusinessException` with namespaced error codes
- All user-facing strings are localized
- Module class uses `[DependsOn]` to declare dependencies
- Middleware configured only in host application
- Never use `.Result` or `.Wait()` on async operations
|-----------|-------------|
| Minimal APIs | ABP Controllers or Auto API Controllers |
| MediatR | Application Services |
| `DbContext` directly in App Services | `IRepository<T>` |
| `AddScoped/AddTransient/AddSingleton` | `ITransientDependency`, `ISingletonDependency` |
| `DateTime.Now` | `IClock` / `Clock.Now` |
| Custom UnitOfWork | ABP's `IUnitOfWorkManager` |
| Manual HTTP calls from UI | ABP client proxies (`generate-proxy`) |
| Hardcoded role checks | Permission-based authorization |
| Business logic in Controllers | Application Services |
