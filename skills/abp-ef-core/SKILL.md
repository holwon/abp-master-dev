---
name: abp-ef-core
description: ABP Entity Framework Core - DbContext, entity configuration, EfCoreRepository implementation, migrations (dotnet ef migrations add), data seeding. Use when working in EntityFrameworkCore projects, adding migrations, or implementing EF Core repositories.
user-invocable: false
---

# ABP Entity Framework Core

ABP provides base classes and conventions for EF Core integration. The EF Core project depends ONLY on the Domain project — never on Application or other layers.

## Never Do

| Don't | Do Instead | Why |
|-------|-----------|-----|
| Skip `b.ConfigureByConvention()` | Always call it first in entity config | Sets up ABP conventions (audit, soft-delete, extra properties) |
| `AddDefaultRepositories(includeAllEntities: true)` | Use `AddDefaultRepositories()` only for aggregate roots | Creates repos for child entities, breaking DDD consistency |
| Inject `DbContext` in application/domain services | Use `IRepository<T>` or custom repository interface | Violates layering, couples to EF Core |
| Use `DbContext` directly outside the EF Core project | Access via `GetDbContextAsync()` inside repository only | Encapsulation, testability |
| Enable lazy loading | Use eager loading via `IncludeDetails` | Lazy loading causes N+1 queries, hard to debug |

## DbContext Configuration

### DbContext Interface (Optional but Recommended for Modules)

```csharp
[ConnectionStringName("Default")]
public interface IMyProjectDbContext : IEfCoreDbContext
{
    DbSet<Book> Books { get; }  // No setters, aggregate roots only
}
```

### DbContext Class

```csharp
[ConnectionStringName("Default")]
public class MyProjectDbContext : AbpDbContext<MyProjectDbContext>, IMyProjectDbContext
{
    public DbSet<Book> Books { get; set; }
    public DbSet<Author> Authors { get; set; }

    // Static table prefix and schema for module extensibility
    public static string TablePrefix { get; set; } = MyProjectConsts.DefaultDbTablePrefix;
    public static string? Schema { get; set; } = MyProjectConsts.DefaultDbSchema;

    public MyProjectDbContext(DbContextOptions<MyProjectDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);  // MUST call base first
        builder.ConfigureMyProject();    // Extension method for entity config
    }
}
```

> Always call `base.OnModelCreating(builder)` FIRST inside the override.

## Entity Configuration

Use extension methods on `ModelBuilder` — never configure entities directly in `OnModelCreating`:

```csharp
public static class MyProjectDbContextModelCreatingExtensions
{
    public static void ConfigureMyProject(this ModelBuilder builder)
    {
        Check.NotNull(builder, nameof(builder));

        builder.Entity<Book>(b =>
        {
            b.ToTable(MyProjectDbContext.TablePrefix + "Books", MyProjectDbContext.Schema);
            b.ConfigureByConvention(); // MUST call for EVERY entity — sets up ABP conventions

            // Property configurations
            b.Property(x => x.Name)
                .IsRequired()
                .HasMaxLength(BookConsts.MaxNameLength);

            b.Property(x => x.Price)
                .HasColumnType("decimal(18,2)");

            // Indexes
            b.HasIndex(x => x.Name);

            // Relationships — reference by ID, not navigation to other aggregate roots
            b.HasOne<Author>()
                .WithMany()
                .HasForeignKey(x => x.AuthorId)
                .OnDelete(DeleteBehavior.Restrict);
        });

        builder.Entity<Author>(b =>
        {
            b.ToTable(MyProjectDbContext.TablePrefix + "Authors", MyProjectDbContext.Schema);
            b.ConfigureByConvention();

            b.Property(x => x.Name).IsRequired().HasMaxLength(AuthorConsts.MaxNameLength);
        });
    }
}
```

### Key Rules for Entity Configuration

- Always call `b.ConfigureByConvention()` for EVERY mapped entity
- Use `TablePrefix` and `Schema` from DbContext static properties (for module extensibility)
- Use constants from `Domain.Shared` for lengths and constraints
- Configure indexes for frequently queried columns
- Use `DeleteBehavior.Restrict` for foreign keys (avoid cascade deletes)

## Repository Implementation

Inherit from `EfCoreRepository<TDbContext, TEntity, TKey>`. Use the DbContext **interface** as the generic parameter:

```csharp
public class BookRepository : EfCoreRepository<IMyProjectDbContext, Book, Guid>, IBookRepository
{
    public BookRepository(IDbContextProvider<IMyProjectDbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }

    public async Task<Book> FindByNameAsync(
        string name,
        bool includeDetails = true,
        CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();

        return await dbSet
            .IncludeDetails(includeDetails)
            .FirstOrDefaultAsync(
                b => b.Name == name,
                GetCancellationToken(cancellationToken));
    }

    public async Task<List<Book>> GetListByAuthorAsync(
        Guid authorId,
        bool includeDetails = false,
        CancellationToken cancellationToken = default)
    {
        var dbSet = await GetDbSetAsync();

        return await dbSet
            .IncludeDetails(includeDetails)
            .Where(b => b.AuthorId == authorId)
            .ToListAsync(GetCancellationToken(cancellationToken));
    }

    public override async Task<IQueryable<Book>> WithDetailsAsync()
    {
        return (await GetQueryableAsync())
            .Include(b => b.Reviews);
    }
}
```

### IncludeDetails Extension Method

Create an extension method per aggregate root for eager loading sub-collections:

```csharp
public static class BookEfCoreQueryableExtensions
{
    public static IQueryable<Book> IncludeDetails(
        this IQueryable<Book> queryable,
        bool include = true)
    {
        if (!include)
        {
            return queryable;
        }

        return queryable
            .Include(b => b.Reviews)
            .Include(b => b.Tags);
    }
}
```

### Repository Best Practices

- Use `GetDbSetAsync()` or `GetQueryableAsync()` to access data (applies ABP data filters)
- Use `GetCancellationToken(cancellationToken)` to pass cancellation tokens
- Override `WithDetailsAsync()` for default eager loading
- Use DbContext **interface** as generic parameter (enables testing with in-memory provider)

## Migration Commands

```bash
# Navigate to EF Core project directory first
cd src/MyProject.EntityFrameworkCore

# Add a new migration
dotnet ef migrations add AddedBookEntity

# Apply migration via DbMigrator (recommended — also seeds data)
dotnet run --project ../MyProject.DbMigrator

# Or apply directly via EF Core CLI
dotnet ef database update

# Remove last migration (if not yet applied to database)
dotnet ef migrations remove

# Generate SQL script for manual deployment
dotnet ef migrations script

# List all migrations
dotnet ef migrations list
```

> ABP templates include `IDesignTimeDbContextFactory` in the EF Core project, so the `-s` (startup project) parameter is NOT needed.

### DbMigrator

The DbMigrator is a console application that:
1. Applies pending migrations
2. Runs data seed contributors (`IDataSeedContributor`)
3. Can be run as part of CI/CD pipeline

## Module Configuration

```csharp
[DependsOn(typeof(AbpEntityFrameworkCoreModule))]
public class MyProjectEntityFrameworkCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<MyProjectDbContext>(options =>
        {
            // Add default repositories for aggregate roots only (DDD best practice)
            options.AddDefaultRepositories();

            // ⚠️ Avoid includeAllEntities: true — it creates repositories for child entities,
            // allowing them to be modified without going through the aggregate root,
            // which breaks data consistency
        });
    }
}
```

## Data Seeding

Implement `IDataSeedContributor` to seed initial data:

```csharp
public class BookStoreDataSeedContributor : IDataSeedContributor, ITransientDependency
{
    private readonly IRepository<Book, Guid> _bookRepository;

    public BookStoreDataSeedContributor(IRepository<Book, Guid> bookRepository)
    {
        _bookRepository = bookRepository;
    }

    public async Task SeedAsync(DataSeedContext context)
    {
        if (await _bookRepository.GetCountAsync() > 0)
        {
            return; // Already seeded
        }

        await _bookRepository.InsertAsync(
            new Book(GuidGenerator.Create(), "The Art of Computer Programming", 99.99m)
        );
    }
}
```

## Database Provider Configuration

ABP supports multiple database providers. Configure in the host application:

```csharp
// SQL Server (default)
context.Services.AddAbpDbContext<MyProjectDbContext>(options =>
{
    options.AddDefaultRepositories();
});

Configure<AbpDbContextOptions>(options =>
{
    options.UseSqlServer();
});

// PostgreSQL
Configure<AbpDbContextOptions>(options =>
{
    options.UseNpgsql();
});

// MySQL
Configure<AbpDbContextOptions>(options =>
{
    options.UseMySQL();
});

// SQLite (for testing)
Configure<AbpDbContextOptions>(options =>
{
    options.UseSqlite();
});
```

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| Skipping `ConfigureByConvention()` | ABP conventions not applied (audit, soft-delete) | Always call it first in entity config |
| `includeAllEntities: true` | Creates repos for child entities | Only add repos for aggregate roots |
| Injecting `DbContext` in Application layer | Violates layering | Use `IRepository<T>` or custom repository |
| Configuring entities in `OnModelCreating` directly | Hard to maintain, not extensible | Use extension methods |
| Enabling lazy loading | N+1 queries, hard to debug | Use `IncludeDetails` for eager loading |
| Not calling `base.OnModelCreating()` | ABP base configurations skipped | Always call base first |
| Hardcoding table names | Not extensible for modules | Use `TablePrefix` and `Schema` static properties |

## Best Practices Checklist

- DbContext inherits from `AbpDbContext<TDbContext>`
- `base.OnModelCreating(builder)` called first
- `b.ConfigureByConvention()` called for every entity
- Entity configuration in extension methods, not in `OnModelCreating`
- `TablePrefix` and `Schema` as static properties
- Repository inherits from `EfCoreRepository<TDbContext, TEntity, TKey>`
- Repository uses DbContext interface as generic parameter
- `IncludeDetails` extension method per aggregate root
- `GetCancellationToken(cancellationToken)` used in all async calls
- `AddDefaultRepositories()` without `includeAllEntities: true`
- Migrations run via DbMigrator (not just `dotnet ef database update`)
- Data seeding via `IDataSeedContributor`

        Configure<AbpDbContextOptions>(options =>
        {
            options.UseSqlServer(); // or UseNpgsql(), UseMySql(), etc.
        });
    }
}
```

## Best Practices

### Repositories for Aggregate Roots Only
Don't use `includeAllEntities: true` in `AddDefaultRepositories()`. This creates repositories for child entities, allowing direct modification without going through the aggregate root - breaking DDD data consistency rules.

```csharp
// ✅ Correct - Only aggregate roots get repositories
options.AddDefaultRepositories();

// ❌ Avoid - Creates repositories for ALL entities including child entities
options.AddDefaultRepositories(includeAllEntities: true);
```

### Always Call ConfigureByConvention
```csharp
builder.Entity<MyEntity>(b =>
{
    b.ConfigureByConvention(); // Don't forget this!
    // Other configurations...
});
```

### Use Table Prefix
```csharp
public static class MyProjectConsts
{
    public const string DbTablePrefix = "App";
    public const string DbSchema = null; // Or "myschema"
}
```

### Performance Tips
- Add explicit indexes for frequently queried fields
- Use `AsNoTracking()` for read-only queries
- Avoid N+1 queries with `.Include()` or specifications
- ABP handles cancellation automatically; use `GetCancellationToken(cancellationToken)` only in custom repository methods
- Consider query splitting for complex queries with multiple collections

### Accessing Raw DbContext
```csharp
public async Task CustomOperationAsync()
{
    var dbContext = await GetDbContextAsync();

    // Raw SQL
    await dbContext.Database.ExecuteSqlRawAsync(
        "UPDATE Books SET IsPublished = 1 WHERE AuthorId = {0}",
        authorId
    );
}
```

## Data Seeding

```csharp
public class MyProjectDataSeedContributor : IDataSeedContributor, ITransientDependency
{
    private readonly IRepository<Book, Guid> _bookRepository;
    private readonly IGuidGenerator _guidGenerator;

    public async Task SeedAsync(DataSeedContext context)
    {
        if (await _bookRepository.GetCountAsync() > 0)
        {
            return;
        }

        await _bookRepository.InsertAsync(
            new Book(_guidGenerator.Create(), "Sample Book", 19.99m, Guid.Empty),
            autoSave: true
        );
    }
}
```
