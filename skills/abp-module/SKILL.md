---
name: abp-module
description: ABP reusable Module solution template - EF Core + MongoDB dual support, virtual methods for extensibility, DbTablePrefix, module options pattern, entity extension, separate connection string. Use when building or reviewing reusable ABP modules that will be distributed or consumed by other solutions.
---

# ABP Module Solution Template

This template is for developing **reusable ABP modules** that can be distributed and consumed by other solutions. The key requirement is **extensibility** — consumers must be able to override and customize module behavior.

## Solution Structure

```
MyModule/
├── src/
│   ├── MyModule.Domain.Shared/      # Constants, enums, localization
│   ├── MyModule.Domain/             # Entities, repository interfaces, domain services
│   ├── MyModule.Application.Contracts/ # DTOs, service interfaces, permissions
│   ├── MyModule.Application/        # Service implementations
│   ├── MyModule.EntityFrameworkCore/ # EF Core implementation
│   ├── MyModule.MongoDB/            # MongoDB implementation
│   ├── MyModule.HttpApi/            # REST controllers
│   ├── MyModule.HttpApi.Client/     # Client proxies (for other services to call)
│   ├── MyModule.Web/                # MVC/Razor Pages UI (optional)
│   └── MyModule.Blazor/             # Blazor UI (optional)
├── test/
│   ├── MyModule.EntityFrameworkCore.Tests/
│   ├── MyModule.MongoDB.Tests/
│   └── MyModule.Application.Tests/
└── host/
    └── MyModule.HttpApi.Host/       # Test host application for development
```

## Database Independence (EF Core + MongoDB)

Modules MUST support both EF Core and MongoDB. Define repository interfaces in Domain, implement in both data layers:

### Repository Interface (Domain)

```csharp
public interface IBookRepository : IRepository<Book, Guid>
{
    Task<Book> FindByNameAsync(
        string name,
        bool includeDetails = true,
        CancellationToken cancellationToken = default);

    Task<List<Book>> GetListByAuthorAsync(
        Guid authorId,
        bool includeDetails = false,
        CancellationToken cancellationToken = default);
}
```

### EF Core Implementation

```csharp
public class BookRepository :
    EfCoreRepository<IMyModuleDbContext, Book, Guid>,
    IBookRepository
{
    public BookRepository(IDbContextProvider<IMyModuleDbContext> dbContextProvider)
        : base(dbContextProvider) { }

    public async Task<Book> FindByNameAsync(
        string name,
        bool includeDetails = true,
        CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();
        return await dbSet
            .IncludeDetails(includeDetails)
            .FirstOrDefaultAsync(b => b.Name == name,
                GetCancellationToken(cancellationToken));
    }
}
```

### MongoDB Implementation

```csharp
public class BookRepository :
    MongoDbRepository<IMyModuleMongoDbContext, Book, Guid>,
    IBookRepository
{
    public BookRepository(IMongoDbContextProvider<IMyModuleMongoDbContext> dbContextProvider)
        : base(dbContextProvider) { }

    public async Task<Book> FindByNameAsync(
        string name,
        bool includeDetails = true,
        CancellationToken cancellationToken = default)
    {
        var queryable = await GetMongoQueryableAsync();
        return await queryable.FirstOrDefaultAsync(
            b => b.Name == name,
            GetCancellationToken(cancellationToken));
    }
}
```

## Table/Collection Prefix

Allow consumers to customize prefixes to avoid naming conflicts:

```csharp
// Domain.Shared
public static class MyModuleDbProperties
{
    public static string DbTablePrefix { get; set; } = "MyModule";
    public static string? DbSchema { get; set; } = null;
    public const string ConnectionStringName = "MyModule";
}
```

Usage in EF Core:

```csharp
builder.Entity<Book>(b =>
{
    b.ToTable(
        MyModuleDbProperties.DbTablePrefix + "Books",
        MyModuleDbProperties.DbSchema
    );
    b.ConfigureByConvention();
});
```

Usage in MongoDB:

```csharp
builder.Entity<Book>(b =>
{
    b.CollectionName = MyModuleDbProperties.DbTablePrefix + "Books";
});
```

## Module Options Pattern

Provide configuration options for consumers:

```csharp
// Domain
public class MyModuleOptions
{
    public bool EnableFeatureX { get; set; } = true;
    public int MaxItemCount { get; set; } = 100;
    public string DefaultCulture { get; set; } = "en";
}
```

Consumer configures in their module:

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    Configure<MyModuleOptions>(options =>
    {
        options.EnableFeatureX = false;
        options.MaxItemCount = 50;
    });
}
```

Usage in service:

```csharp
public class MyService : ITransientDependency
{
    private readonly MyModuleOptions _options;

    public MyService(IOptions<MyModuleOptions> options)
    {
        _options = options.Value;
    }

    public void DoSomething()
    {
        if (_options.EnableFeatureX)
        {
            // Feature-specific logic
        }
    }
}
```

## Extensibility — Virtual Methods (CRITICAL!)

When developing a reusable module, **ALL public and protected methods MUST be `virtual`** to allow consumers to override behavior:

```csharp
public class BookAppService : ApplicationService, IBookAppService
{
    // ✅ Public methods MUST be virtual
    public virtual async Task<BookDto> CreateAsync(CreateBookDto input)
    {
        var book = await CreateBookEntityAsync(input);
        await _bookRepository.InsertAsync(book);
        return ObjectMapper.Map<Book, BookDto>(book);
    }

    // ✅ Use protected virtual for helper methods (NOT private)
    protected virtual Task<Book> CreateBookEntityAsync(CreateBookDto input)
    {
        return Task.FromResult(new Book(
            GuidGenerator.Create(),
            input.Name,
            input.Price
        ));
    }

    // ❌ WRONG for modules — private methods cannot be overridden
    // private Book CreateBook(CreateBookDto input) { ... }
}
```

This allows module consumers to:
- Override specific methods without copying entire class
- Extend functionality while preserving base behavior via `base.Method()`
- Customize module behavior for their specific needs

## Entity Extension

Support the object extension system so consumers can add properties:

```csharp
public static class MyModuleModuleExtensionConfigurator
{
    private static readonly OneTimeRunner Runner = new();

    public static void Configure()
    {
        Runner.Run(() =>
        {
            ObjectExtensionManager.Instance.Modules()
                .ConfigureMyModule(module =>
                {
                    module.ConfigureBook(book =>
                    {
                        book.AddOrUpdateProperty<string>(
                            "CustomProperty",
                            options =>
                            {
                                options.DefaultValue = "Default";
                                options.Attributes.Add(
                                    new StringLengthAttribute(128)
                                );
                            }
                        );
                    });
                });
        });
    }
}
```

## Localization

```csharp
// Domain.Shared
[LocalizationResourceName("MyModule")]
public class MyModuleResource
{
}

// Module configuration
Configure<AbpLocalizationOptions>(options =>
{
    options.Resources
        .Add<MyModuleResource>("en")
        .AddVirtualJson("/Localization/MyModule");
});
```

## Permission Definition

```csharp
public class MyModulePermissionDefinitionProvider : PermissionDefinitionProvider
{
    public override void Define(IPermissionDefinitionContext context)
    {
        var myGroup = context.AddGroup(
            MyModulePermissions.GroupName,
            L("Permission:MyModule"));

        myGroup.AddPermission(
            MyModulePermissions.Books.Default,
            L("Permission:Books"));
    }

    private static LocalizableString L(string name)
    {
        return LocalizableString.Create<MyModuleResource>(name);
    }
}
```

## Best Practices

1. **Virtual methods** — All public/protected methods must be `virtual` for extensibility
2. **Protected virtual helpers** — Use `protected virtual` instead of `private` for helper methods
3. **Database agnostic** — Support both EF Core and MongoDB with dual repository implementations
4. **Configurable** — Use options pattern (`MyModuleOptions`) for customization
5. **Localizable** — Use localization for all user-facing text
6. **Table prefix** — Allow customization via `DbTablePrefix` to avoid conflicts
7. **Separate connection string** — Support dedicated database via `[ConnectionStringName]`
8. **No dependencies on host** — Module should be self-contained
9. **Test with host app** — Include a host application for development and testing
10. **Entity extension support** — Use `ObjectExtensionManager` for consumer extensibility

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| Non-virtual public methods | Consumers can't override behavior | Make all public/protected methods `virtual` |
| `private` helper methods | Can't be overridden by consumers | Use `protected virtual` |
| Hardcoded table names | Conflicts with consumer's tables | Use `DbTablePrefix` and `DbSchema` |
| Single DB provider only | Limits consumer choice | Support both EF Core and MongoDB |
| Hardcoded configuration values | Not customizable | Use options pattern |
| Missing localization | Not usable in multi-language apps | Localize all user-facing strings |

## Best Practices Checklist

- [ ] All public/protected methods are `virtual`
- [ ] Helper methods use `protected virtual` (not `private`)
- [ ] Repository interfaces in Domain, implementations in both EF Core and MongoDB
- [ ] `DbTablePrefix` and `DbSchema` as configurable static properties
- [ ] `[ConnectionStringName]` for separate database support
- [ ] `MyModuleOptions` for configuration
- [ ] `ObjectExtensionManager` for entity extension
- [ ] Localization resource defined in Domain.Shared
- [ ] Permission definitions in Application.Contracts
- [ ] Test host application included for development
