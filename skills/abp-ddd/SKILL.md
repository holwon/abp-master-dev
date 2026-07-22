---
name: abp-ddd
description: ABP DDD patterns - Entities, Aggregate Roots, value objects, Repositories, Domain Services, Domain Events, Specifications. Use when designing domain layer, creating entities, repositories, or domain services in ABP projects.
user-invocable: false
---

# ABP Domain-Driven Design (DDD)

ABP provides base classes, interfaces, and conventions to implement DDD patterns. The framework encourages **Rich Domain Models** but does not force any specific DDD rule — it makes DDD easier when you choose to apply it.

## Entity Class Hierarchy

ABP provides a hierarchy of base classes for entities and aggregate roots:

### Entity Base Classes

| Base Class | Key Type | Includes | Use When |
|-----------|---------|----------|----------|
| `Entity<TKey>` | Single (Guid, int, string, long) | `Id` property | Simple entities, sub-entities |
| `Entity` | Composite keys | `GetKeys()` method | Entities with composite PKs |

### Aggregate Root Base Classes

| Base Class | Key Type | Includes | Use When |
|-----------|---------|----------|----------|
| `AggregateRoot<TKey>` | Single | `Id` + `ExtraProperties` + `ConcurrencyStamp` | Aggregate roots needing extensibility & optimistic concurrency |
| `BasicAggregateRoot<TKey>` | Single | `Id` only | Aggregate roots not needing extra properties or concurrency stamp |
| `AggregateRoot` | Composite keys | Same as `AggregateRoot<TKey>` | Aggregate roots with composite PKs (rare, not recommended) |
| `BasicAggregateRoot` | Composite keys | `Id` only | Minimal aggregate roots with composite PKs |

### Audited Base Classes

ABP automatically sets audit properties when you use these base classes. The `...WithUser` variants (e.g., `FullAuditedAggregateRootWithUser<TUser>`) add a navigation property to the user entity — but this is **not recommended** for DDD as it breaks aggregate boundaries.

| Base Class | Auto-Set Properties |
|-----------|-------------------|
| `CreationAuditedEntity<TKey>` | `CreationTime`, `CreatorId` |
| `CreationAuditedAggregateRoot<TKey>` | `CreationTime`, `CreatorId` |
| `AuditedEntity<TKey>` | `CreationTime`, `CreatorId`, `LastModificationTime`, `LastModifierId` |
| `AuditedAggregateRoot<TKey>` | `CreationTime`, `CreatorId`, `LastModificationTime`, `LastModifierId` |
| `FullAuditedEntity<TKey>` | All above + `IsDeleted`, `DeletionTime`, `DeleterId` (soft-delete) |
| `FullAuditedAggregateRoot<TKey>` | All above + `IsDeleted`, `DeletionTime`, `DeleterId` (soft-delete) |

> `FullAuditedAggregateRoot` implements `ISoftDelete` — the entity is soft-deleted, not physically removed. ABP's `DataFilter` automatically excludes soft-deleted entities from queries.

### Audit Interfaces (for manual implementation)

If you prefer implementing interfaces instead of inheriting base classes:

| Interface | Properties |
|-----------|-----------|
| `IHasCreationTime` | `CreationTime` |
| `IMayHaveCreator` | `CreatorId` |
| `ICreationAuditedObject` | `CreationTime` + `CreatorId` |
| `IHasModificationTime` | `LastModificationTime` |
| `IModificationAuditedObject` | `LastModificationTime` + `LastModifierId` |
| `IAuditedObject` | Creation + Modification audit |
| `ISoftDelete` | `IsDeleted` (enables soft-delete filter) |
| `IHasDeletionTime` | `IsDeleted` + `DeletionTime` |
| `IDeletionAuditedObject` | `IsDeleted` + `DeletionTime` + `DeleterId` |
| `IFullAuditedObject` | Creation + Modification + Deletion audit |

## Entity Design Best Practices

### 1. GUID Keys — Use IGuidGenerator, Never Guid.NewGuid()

`IGuidGenerator` produces **sequential GUIDs** optimized for clustered indexes in relational databases. Never call `Guid.NewGuid()` inside an entity constructor.

```csharp
public class Book : Entity<Guid>
{
    public string Name { get; protected set; }
    public float Price { get; protected set; }

    protected Book() { }  // Required for ORM deserialization

    public Book(Guid id, string name, float price) : base(id)
    {
        Name = name;
        Price = price;
    }
}

// In Application Service — GuidGenerator is a pre-injected property:
var book = new Book(GuidGenerator.Create(), input.Name, input.Price);
await _bookRepository.InsertAsync(book);
```

### 2. Encapsulate Properties — Use Protected/Private Setters

Properties should have `protected set` or `private set` to prevent external mutation. State changes happen through domain methods with meaningful names:

```csharp
public class Order : AggregateRoot<Guid>
{
    public string ReferenceNo { get; protected set; }
    public int TotalItemCount { get; protected set; }
    public OrderStatus Status { get; protected set; }
    public List<OrderLine> OrderLines { get; protected set; }

    protected Order() { }

    public Order(Guid id, string referenceNo) : base(id)
    {
        ReferenceNo = Check.NotNull(referenceNo, nameof(referenceNo));
        Status = OrderStatus.Created;
        OrderLines = new List<OrderLine>();
    }

    public void AddProduct(Guid productId, int count)
    {
        if (count <= 0)
            throw new ArgumentException("Count must be positive", nameof(count));

        if (Status != OrderStatus.Created)
            throw new BusinessException("Orders:CannotModifyOrder");

        var existing = OrderLines.FirstOrDefault(ol => ol.ProductId == productId);
        if (existing == null)
            OrderLines.Add(new OrderLine(Id, productId, count));
        else
            existing.ChangeCount(existing.Count + count);

        TotalItemCount += count;
    }

    public void Complete()
    {
        if (Status != OrderStatus.Created)
            throw new BusinessException("Orders:CannotCompleteOrder");

        Status = OrderStatus.Completed;
        AddLocalEvent(new OrderCompletedEvent { OrderId = Id });
    }
}
```

### 3. Sub-Entities — Internal Constructors

Sub-entities (non-aggregate-root entities) should have `internal` constructors so they can only be created by the aggregate root within the domain layer:

```csharp
public class OrderLine : Entity
{
    public Guid OrderId { get; protected set; }
    public Guid ProductId { get; protected set; }
    public int Count { get; protected set; }

    protected OrderLine() { }

    internal OrderLine(Guid orderId, Guid productId, int count)
    {
        OrderId = orderId;
        ProductId = productId;
        Count = count;
    }

    internal void ChangeCount(int newCount) => Count = newCount;

    public override object[] GetKeys() => new object[] { OrderId, ProductId };
}
```

### 4. Composite Keys — Override GetKeys()

```csharp
public class UserRole : Entity
{
    public Guid UserId { get; set; }
    public Guid RoleId { get; set; }

    public override object[] GetKeys() => new object[] { UserId, RoleId };
}
```

> Entities with composite keys cannot use `IRepository<TEntity, TKey>`. Use `IRepository<TEntity>` instead.

### 5. Always Initialize Collections in Constructor

```csharp
public Order(Guid id, string referenceNo) : base(id)
{
    OrderLines = new List<OrderLine>();  // Always initialize
}
```

### 6. Entity Equality — EntityEquals

ABP provides `EntityEquals()` for comparing entities:

```csharp
Book book1 = ...;
Book book2 = ...;
if (book1.EntityEquals(book2)) { /* same entity */ }
```

### 7. IKeyedObject Interface

All ABP entities implement `IKeyedObject`, which provides `GetObjectKey()` — returns the primary key as a string. Useful for resource-based authorization, caching, and logging.

## Aggregate Root Rules

1. **Reference by ID only** — Never add navigation properties to other aggregate roots
2. **Single transaction boundary** — An aggregate is loaded and saved as a unit
3. **One repository per aggregate root** — No repositories for sub-entities
4. **Preserve invariants** — The aggregate root is responsible for the consistency of its entire cluster
5. **Keep aggregates small** — Avoid large sub-collections; consider splitting if needed
6. **Work with sub-entities through the root** — Never modify sub-entities independently

## Value Objects

Value objects have no identity — they are defined by their attributes. ABP provides `ValueObject<T>` base class:

```csharp
public class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string PostalCode { get; }

    public Address(string street, string city, string postalCode)
    {
        Street = street;
        City = city;
        PostalCode = postalCode;
    }

    protected override IEnumerable<object> GetAtomicValues()
    {
        yield return Street;
        yield return City;
        yield return PostalCode;
    }
}
```

Override `GetAtomicValues()` to define which properties determine equality.

## Repositories

### When to Create a Custom Repository

Create a custom repository interface when:
- You need query methods beyond basic CRUD
- You need domain-specific data access logic
- The default `IRepository<TEntity, TKey>` is insufficient

### Repository Interface (in Domain layer)

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

### Method Conventions

- All methods are `async` and return `Task` or `Task<T>`
- Include `CancellationToken cancellationToken = default` in every method
- Single-entity methods: `bool includeDetails = true` (default)
- List methods: `bool includeDetails = false` (default)
- Do NOT return composite projection classes — use `includeDetails` for eager loading
- Do NOT expose `IQueryable` in **public** repository interfaces (internal implementations may use it)

### Repository Implementation (in EF Core / MongoDB layer)

```csharp
public class BookRepository : EfCoreRepository<BookStoreDbContext, Book, Guid>, IBookRepository
{
    public BookRepository(IDbContextProvider<BookStoreDbContext> dbContextProvider)
        : base(dbContextProvider) { }

    public async Task<Book> FindByNameAsync(
        string name,
        bool includeDetails = true,
        CancellationToken cancellationToken = default)
    {
        var dbContext = await GetDbContextAsync();
        return await dbContext.Books
            .IncludeDetails(includeDetails)
            .FirstOrDefaultAsync(b => b.Name == name, GetCancellationToken(cancellationToken));
    }

    public async Task<List<Book>> GetListByAuthorAsync(
        Guid authorId,
        bool includeDetails = false,
        CancellationToken cancellationToken = default)
    {
        var dbContext = await GetDbContextAsync();
        return await dbContext.Books
            .IncludeDetails(includeDetails)
            .Where(b => b.AuthorId == authorId)
            .ToListAsync(GetCancellationToken(cancellationToken));
    }
}
```

### Repository Best Practices

- **One repository per aggregate root only** — Never create repositories for child entities
- Child entities must be accessed/modified only through their aggregate root
- Creating repositories for child entities breaks data consistency (bypasses aggregate root's business rules)
- In ABP, use `AddDefaultRepositories()` without `includeAllEntities: true` to enforce this
- Define custom repository only when custom queries are needed
- Interface in Domain layer, implementation in data layer (EF Core / MongoDB)

```csharp
// ✅ Correct: Repository for aggregate root (Order)
public interface IOrderRepository : IRepository<Order, Guid> { }

// ❌ Wrong: Repository for child entity (OrderLine)
// OrderLine should only be accessed through Order aggregate
public interface IOrderLineRepository : IRepository<OrderLine, Guid> { } // Don't do this!
```

## Domain Services

Domain services encapsulate business logic that doesn't naturally fit in a single entity or aggregate.

### When to Use a Domain Service

- Logic depends on multiple aggregates/entities
- Logic requires external services (repositories, etc.)
- The operation doesn't belong to any single entity

### Conventions

- Name with `Manager` suffix (e.g., `IssueManager`, `OrderManager`)
- Derive from `DomainService` base class (auto-registers as **Transient**)
- Do NOT create interfaces unless needed for mocking/testing
- Accept domain objects (entities), NOT DTOs
- Throw `BusinessException` for rule violations with namespaced error codes
- Do NOT depend on `ICurrentUser` — pass user ID from the application layer

```csharp
public class IssueManager : DomainService
{
    private readonly IRepository<Issue, Guid> _issueRepository;

    public IssueManager(IRepository<Issue, Guid> issueRepository)
    {
        _issueRepository = issueRepository;
    }

    public async Task AssignAsync(Issue issue, IdentityUser user)
    {
        var currentCount = await _issueRepository
            .CountAsync(i => i.AssignedUserId == user.Id);

        if (currentCount >= 3)
            throw new BusinessException("IssueTracking:ConcurrentOpenIssueLimit");

        issue.AssignTo(user.Id);
    }
}
```

### DomainService Base Properties

When you derive from `DomainService`, you get these pre-injected properties:
- `GuidGenerator` (`IGuidGenerator`)
- `Clock` (`IClock`)
- `LoggerFactory` (`ILoggerFactory`)
- `LazyServiceProvider` (`ILazyServiceProvider`)

## Specifications

Specifications encapsulate query logic in reusable, composable classes. Define in the Domain layer:

```csharp
public class ActiveProductSpecification : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
    {
        return p => p.IsActive && p.StockCount > 0;
    }
}

// Usage in repository or application service:
var query = await _productRepository.GetQueryableAsync();
var activeProducts = await query
    .Where(new ActiveProductSpecification())
    .ToListAsync();
```

## Unit of Work

ABP automatically manages the unit of work for application services. Every application service method is wrapped in a `UnitOfWork` scope:

- Changes are committed at the end of the method (if no exception)
- Rolled back automatically on exception
- Use `[UnitOfWork(isTransactional: false)]` for read-only operations
- Use `[UnitOfWork]` attribute to control scope on non-service methods

## Domain Events

### Local Events (same process, same transaction)

Entities publish events; handlers run in the same transaction:

```csharp
// Entity publishes event:
public class Order : AggregateRoot<Guid>
{
    public void Complete()
    {
        Status = OrderStatus.Completed;
        AddLocalEvent(new OrderCompletedEvent { OrderId = Id });
    }
}

// Handler (in Application layer):
public class OrderCompletedEventHandler : ILocalEventHandler<OrderCompletedEvent>
{
    public async Task HandleEventAsync(OrderCompletedEvent eventData)
    {
        // Handle the event — runs in same UoW transaction
    }
}
```

### Distributed Events (cross-service, async)

Define ETOs (Event Transfer Objects) in `*.Domain.Shared`:

```csharp
// ETO in Domain.Shared:
public class OrderCompletedEto
{
    public Guid OrderId { get; set; }
    public string ReferenceNo { get; set; }
    public decimal TotalAmount { get; set; }
}

// Entity publishes:
AddDistributedEvent(new OrderCompletedEto { OrderId = Id, ReferenceNo = ReferenceNo });

// Handler (in another service's Application layer):
public class OrderCompletedEventHandler : IDistributedEventHandler<OrderCompletedEto>
{
    public async Task HandleEventAsync(OrderCompletedEto eventData)
    {
        // Handle cross-service event
    }
}
```

## Extra Properties

`AggregateRoot<TKey>` implements `IHasExtraProperties`, allowing dynamic property storage:

```csharp
// Set a property:
user.SetProperty("Title", "Manager");

// Get a property:
var title = user.GetProperty<string>("Title");

// Check existence:
if (user.HasProperty("Title")) { ... }

// Remove:
user.RemoveProperty("Title");
```

Best practice: define extension methods with constants for type safety:

```csharp
public static class IdentityUserExtensions
{
    private const string TitlePropertyName = "Title";

    public static void SetTitle(this IdentityUser user, string title)
        => user.SetProperty(TitlePropertyName, title);

    public static string GetTitle(this IdentityUser user)
        => user.GetProperty<string>(TitlePropertyName);
}
```

> For EF Core, extra properties are stored as JSON in a single `ExtraProperties` column by default. For MongoDB, they're stored as regular document fields.

## Entity Versioning

Implement `IHasEntityVersion` for automatic version tracking:

```csharp
public interface IHasEntityVersion
{
    int EntityVersion { get; }
}
```

ABP automatically increments `EntityVersion` on each update. Initial value is `0`.

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| Anemic entities (public get/set, no methods) | Business logic scattered in services | Encapsulate state, expose domain methods |
| `Guid.NewGuid()` in entity constructor | Non-sequential GUIDs hurt DB performance | Use `IGuidGenerator` from calling code |
| Navigation properties to other aggregate roots | Breaks aggregate boundaries, causes lazy-load issues | Reference by ID only |
| Repository for sub-entities | Sub-entities should be accessed through aggregate root | One repository per aggregate root |
| Domain service depending on `ICurrentUser` | Couples domain to web/auth concerns | Pass user ID from application layer |
| Business logic in application services | Violates DDD layering | Move to domain entities or domain services |
| `IQueryable` in public repository interface | Leaks data access abstraction | Encapsulate queries in repository methods |
| Composite projection classes from repositories | Mixes data access with presentation concerns | Use `includeDetails` for eager loading |

## Best Practices Checklist

- Entities inherit from appropriate base class (`Entity<TKey>`, `AggregateRoot<TKey>`, etc.)
- GUID keys generated via `IGuidGenerator`, not `Guid.NewGuid()`
- Properties have `protected`/`private` setters
- State mutations happen through domain methods with meaningful names
- Collections initialized in constructor
- Sub-entities have `internal` constructors
- Aggregate roots reference other aggregates by ID only
- One repository per aggregate root
- Repository interfaces in Domain layer, implementations in EF Core/MongoDB layer
- Domain services named with `Manager` suffix
- Domain services throw `BusinessException` with namespaced error codes
- Domain events used for side effects within the same aggregate
- Value objects override `GetAtomicValues()` for equality
- Protected parameterless constructor present for ORM deserialization
- Accept/return domain objects, not DTOs
- Don't depend on authenticated user - pass values from application layer
- Use base class properties (`GuidGenerator`, `Clock`) instead of injecting these services

## Domain Events

### Local Events
```csharp
// In aggregate
AddLocalEvent(new OrderCompletedEvent(Id));

// Handler
public class OrderCompletedEventHandler : ILocalEventHandler<OrderCompletedEvent>, ITransientDependency
{
    public async Task HandleEventAsync(OrderCompletedEvent eventData)
    {
        // Handle within same transaction
    }
}
```

### Distributed Events (ETO)
For inter-module/microservice communication:
```csharp
// In Domain.Shared
[EventName("Orders.OrderCompleted")]
public class OrderCompletedEto
{
    public Guid OrderId { get; set; }
    public string OrderNumber { get; set; }
}
```

## Specifications

Reusable query conditions:
```csharp
public class CompletedOrdersSpec : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
    {
        return o => o.Status == OrderStatus.Completed;
    }
}

// Usage
var orders = await _orderRepository.GetListAsync(new CompletedOrdersSpec());
```
