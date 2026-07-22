---
name: abp-mongodb
description: ABP MongoDB patterns - AbpMongoDbContext, IMongoCollection, MongoDbRepository, no migrations, embedded documents vs references, manual UpdateAsync required. Use when working in MongoDB projects or implementing MongoDB repositories in ABP.
user-invocable: false
---

# ABP MongoDB

ABP provides base classes and conventions for MongoDB integration. MongoDB is schema-less — no migrations needed. The MongoDB project depends ONLY on the Domain project.

## MongoDbContext Configuration

### MongoDbContext Interface (Optional but Recommended for Modules)

```csharp
[ConnectionStringName("Default")]
public interface IMyProjectMongoDbContext : IAbpMongoDbContext
{
    IMongoCollection<Book> Books { get; }  // Aggregate roots only
}
```

### MongoDbContext Class

```csharp
[ConnectionStringName("Default")]
public class MyProjectMongoDbContext : AbpMongoDbContext, IMyProjectMongoDbContext
{
    public IMongoCollection<Book> Books => Collection<Book>();
    public IMongoCollection<Author> Authors => Collection<Author>();

    // Static collection prefix for module extensibility
    public static string CollectionPrefix { get; set; } = MyProjectConsts.DefaultDbTablePrefix;

    protected override void CreateModel(IMongoModelBuilder modelBuilder)
    {
        base.CreateModel(modelBuilder);  // MUST call base first
        modelBuilder.ConfigureMyProject(); // Extension method for entity config
    }
}
```

> Always call `base.CreateModel(modelBuilder)` FIRST inside the override.

## Entity Configuration

Use extension methods on `IMongoModelBuilder` — never configure entities directly in `CreateModel`:

```csharp
public static class MyProjectMongoDbContextExtensions
{
    public static void ConfigureMyProject(this IMongoModelBuilder builder)
    {
        Check.NotNull(builder, nameof(builder));

        builder.Entity<Book>(b =>
        {
            b.CollectionName = MyProjectMongoDbContext.CollectionPrefix + "Books";

            // Configure indexes
            b.HasIndex(x => x.Name);
            b.HasIndex(x => x.AuthorId);
        });

        builder.Entity<Author>(b =>
        {
            b.CollectionName = MyProjectMongoDbContext.CollectionPrefix + "Authors";
        });
    }
}
```

### Index Configuration

MongoDB requires explicit index configuration for query performance:

```csharp
builder.Entity<Book>(b =>
{
    b.CollectionName = "Books";

    // Single field index
    b.HasIndex(x => x.Name);

    // Compound index
    b.HasIndex(x => new { x.AuthorId, x.PublishDate });

    // Unique index
    b.HasIndex(x => x.Isbn, unique: true);
});
```

## Repository Implementation

Inherit from `MongoDbRepository<TMongoDbContext, TEntity, TKey>`. Use the MongoDbContext **interface** as the generic parameter:

```csharp
public class BookRepository : MongoDbRepository<IMyProjectMongoDbContext, Book, Guid>, IBookRepository
{
    public BookRepository(IMongoDbContextProvider<IMyProjectMongoDbContext> dbContextProvider)
        : base(dbContextProvider)
    {
    }

    public async Task<Book> FindByNameAsync(
        string name,
        bool includeDetails = true,
        CancellationToken cancellationToken = default)
    {
        var queryable = await GetMongoQueryableAsync();

        return await queryable
            .FirstOrDefaultAsync(
                b => b.Name == name,
                GetCancellationToken(cancellationToken));
    }

    public async Task<List<Book>> GetListByAuthorAsync(
        Guid authorId,
        bool includeDetails = false,
        CancellationToken cancellationToken = default)
    {
        var queryable = await GetMongoQueryableAsync();

        return await queryable
            .Where(b => b.AuthorId == authorId)
            .ToListAsync(GetCancellationToken(cancellationToken));
    }
}
```

### Repository Best Practices

- Use `GetMongoQueryableAsync()` to access data (applies ABP data filters)
- Use `GetCancellationToken(cancellationToken)` to pass cancellation tokens
- Use MongoDbContext **interface** as generic parameter (enables testing)
- `includeDetails` is often ignored in MongoDB — documents already embed related data

## Module Configuration

```csharp
[DependsOn(typeof(AbpMongoDbModule))]
public class MyProjectMongoDbModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddMongoDbContext<MyProjectMongoDbContext>(options =>
        {
            // Add default repositories for aggregate roots only (DDD best practice)
            options.AddDefaultRepositories();

            // ⚠️ Avoid includeAllEntities: true — breaks DDD data consistency
        });
    }
}
```

## Connection String

In `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "Default": "mongodb://localhost:27017/MyProjectDb"
  }
}
```

For MongoDB Atlas or replica sets:

```json
{
  "ConnectionStrings": {
    "Default": "mongodb+srv://user:password@cluster.mongodb.net/MyProjectDb?retryWrites=true&w=majority"
  }
}
```

## Key Differences from EF Core

### 1. No Migrations

MongoDB is schema-less — no migrations needed. Changes to entity structure are handled automatically. Add new properties to your C# entity class and they're immediately available. Remove properties and they're simply ignored in existing documents.

### 2. includeDetails Parameter

Often ignored in MongoDB because documents typically embed related data:

```csharp
public async Task<List<Book>> GetListAsync(
    bool includeDetails = false, // Usually ignored in MongoDB
    CancellationToken cancellationToken = default)
{
    // MongoDB documents already include nested/embedded data
    return await (await GetMongoQueryableAsync())
        .ToListAsync(GetCancellationToken(cancellationToken));
}
```

### 3. Embedded Documents vs References

MongoDB supports two modeling approaches:

```csharp
// Embedded (stored in same document) — good for contained data
public class Order : AggregateRoot<Guid>
{
    public List<OrderLine> Lines { get; set; } // Embedded sub-collection
}

// Reference (separate collection, store ID only) — good for shared data
public class Order : AggregateRoot<Guid>
{
    public Guid CustomerId { get; set; } // Reference by ID
}
```

**Guidelines**:
- **Embed** when: data is contained within the aggregate, always accessed together, has a "has-a" relationship
- **Reference** when: data is shared across aggregates, independently accessed, or large/unbounded

### 4. No Change Tracking — Must Call UpdateAsync Explicitly

MongoDB doesn't track entity changes automatically. You MUST call `UpdateAsync` after mutations:

```csharp
public async Task UpdateBookAsync(Guid id, string newName)
{
    var book = await _bookRepository.GetAsync(id);
    book.SetName(newName);

    // Must explicitly update — MongoDB has no change tracker
    await _bookRepository.UpdateAsync(book);
}
```

> This is different from EF Core where `UpdateAsync` may not be needed if change tracking is enabled. In MongoDB, it's ALWAYS required.

### 5. Direct Collection Access

For operations not covered by the repository, access the MongoDB collection directly:

```csharp
public async Task BulkUpdatePricesAsync(Guid authorId, decimal newPrice)
{
    var collection = await GetCollectionAsync();

    var filter = Builders<Book>.Filter.Eq(b => b.AuthorId, authorId);
    var update = Builders<Book>.Update.Set(b => b.Price, newPrice);

    await collection.UpdateManyAsync(filter, update);
}
```

### 6. Extra Properties Storage

In MongoDB, `ExtraProperties` are stored as **regular document fields** (not JSON string like EF Core). MongoDB natively supports dynamic elements.

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| Not calling `UpdateAsync` after mutation | Changes are lost — MongoDB has no change tracker | Always call `UpdateAsync` explicitly |
| `includeAllEntities: true` | Creates repos for child entities | Only add repos for aggregate roots |
| Embedding unbounded collections | Document size grows indefinitely | Reference by ID for large/unbounded collections |
| Embedding shared/referenced data | Data duplication, inconsistency | Reference by ID for shared aggregates |
| Not configuring indexes | Poor query performance | Configure indexes for frequently queried fields |
| Using EF Core patterns (migrations, change tracking) | MongoDB works differently | Understand MongoDB's document model |

## Best Practices Checklist

- MongoDbContext inherits from `AbpMongoDbContext`
- `base.CreateModel(modelBuilder)` called first
- Entity configuration in extension methods, not in `CreateModel`
- `CollectionPrefix` as static property for module extensibility
- Repository inherits from `MongoDbRepository<TDbContext, TEntity, TKey>`
- Repository uses MongoDbContext interface as generic parameter
- `GetMongoQueryableAsync()` used for data access (applies ABP filters)
- `GetCancellationToken(cancellationToken)` used in all async calls
- `UpdateAsync` called explicitly after every mutation
- Indexes configured for frequently queried fields
- Embedded vs reference decision made consciously per relationship
- `AddDefaultRepositories()` without `includeAllEntities: true`

    await collection.UpdateManyAsync(filter, update);
}
```

## Indexing

Configure indexes in repository or via MongoDB driver:

```csharp
public class BookRepository : MongoDbRepository<MyProjectMongoDbContext, Book, Guid>, IBookRepository
{
    public override async Task<IQueryable<Book>> GetQueryableAsync()
    {
        var collection = await GetCollectionAsync();

        // Ensure index exists
        var indexKeys = Builders<Book>.IndexKeys.Ascending(b => b.Name);
        await collection.Indexes.CreateOneAsync(new CreateIndexModel<Book>(indexKeys));

        return await base.GetQueryableAsync();
    }
}
```

## Best Practices

- Design documents for query patterns (denormalize when needed)
- Use references for frequently changing data
- Use embedding for data that's always accessed together
- Add indexes for frequently queried fields
- Use `GetCancellationToken(cancellationToken)` for proper cancellation
- Remember: ABP data filters (soft-delete, multi-tenancy) work with MongoDB too
