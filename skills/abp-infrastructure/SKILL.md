---
name: abp-infrastructure
description: ABP infrastructure services - ISettingProvider, IFeatureChecker, IDistributedCache, ILocalEventBus, IDistributedEventBus, IBackgroundJobManager, localization resource. Use when working with settings, feature flags, caching, event bus, or background jobs in ABP.
user-invocable: false
---

# ABP Infrastructure Services

ABP provides built-in infrastructure services for settings, features, caching, event bus, background jobs, distributed locking, and more. These services are available across all layers.

## Settings

Settings store configuration values that can be changed at runtime by tenants or users.

### Define Settings

```csharp
public class MySettingDefinitionProvider : SettingDefinitionProvider
{
    public override void Define(ISettingDefinitionContext context)
    {
        context.Add(
            new SettingDefinition("MyApp.MaxItemCount", "10"),
            new SettingDefinition("MyApp.EnableFeature", "false"),
            new SettingDefinition("MyApp.SecretKey", isEncrypted: true),
            new SettingDefinition(
                "MyApp.AllowedDomains",
                defaultValue: "example.com",
                isVisibleToClients: false  // Hidden from client-side
            )
        );
    }
}
```

### Read Settings

```csharp
public class MyService : ITransientDependency
{
    private readonly ISettingProvider _settingProvider;

    public MyService(ISettingProvider settingProvider)
    {
        _settingProvider = settingProvider;
    }

    public async Task DoSomethingAsync()
    {
        var maxCount = await _settingProvider.GetAsync<int>("MyApp.MaxItemCount");
        var isEnabled = await _settingProvider.IsTrueAsync("MyApp.EnableFeature");
        var secret = await _settingProvider.GetOrNullAsync("MyApp.SecretKey");
    }
}
```

### Setting Value Provider Priority (Highest to Lowest)

1. **User settings** — Per-user overrides (highest priority)
2. **Tenant settings** — Per-tenant overrides
3. **Global settings** — System-wide values
4. **Configuration** — `appsettings.json` values
5. **Default value** — Defined in `SettingDefinition` (lowest priority)

### Setting Management

```csharp
public class MyService : ITransientDependency
{
    private readonly ISettingManager _settingManager;

    public async Task UpdateSettingAsync()
    {
        // Set global value
        await _settingManager.SetGlobalAsync("MyApp.MaxItemCount", "20");

        // Set for specific tenant
        await _settingManager.SetForTenantAsync(tenantId, "MyApp.MaxItemCount", "15");

        // Set for specific user
        await _settingManager.SetForUserAsync(userId, "MyApp.MaxItemCount", "5");
    }
}
```

## Features

Features control which functionality is available per tenant (SaaS multi-tenancy).

### Define Features

```csharp
public class MyFeatureDefinitionProvider : FeatureDefinitionProvider
{
    public override void Define(IFeatureDefinitionContext context)
    {
        var myGroup = context.AddGroup("MyApp");

        myGroup.AddFeature(
            "MyApp.PdfReporting",
            defaultValue: "false",
            displayName: L("Feature:PdfReporting"),
            valueType: new ToggleStringValueType()
        );

        myGroup.AddFeature(
            "MyApp.MaxProductCount",
            defaultValue: "10",
            displayName: L("Feature:MaxProductCount"),
            valueType: new FreeTextStringValueType(
                new NumericValueValidator(1, 1000)
            )
        );
    }
}
```

### Check Features

```csharp
// Declarative — throws if feature not enabled
[RequiresFeature("MyApp.PdfReporting")]
public async Task<PdfReportDto> GetPdfReportAsync()
{
    // Only executes if feature is enabled for current tenant
}

// Programmatic — check without throwing
public class MyService : ApplicationService
{
    public async Task DoSomethingAsync()
    {
        if (await FeatureChecker.IsEnabledAsync("MyApp.PdfReporting"))
        {
            // Feature is enabled for current tenant
        }

        var maxCount = await FeatureChecker.GetAsync<int>("MyApp.MaxProductCount");
    }
}
```

> `FeatureChecker` is available as a property in `ApplicationService` and `AbpController` base classes.

## Distributed Caching

ABP provides typed distributed caching with automatic cache invalidation.

### Typed Cache Item

```csharp
[CacheName("Books")]
public class BookCacheItem
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string AuthorName { get; set; }
}
```

### Using Distributed Cache

```csharp
public class BookService : ITransientDependency
{
    private readonly IDistributedCache<BookCacheItem> _cache;

    public BookService(IDistributedCache<BookCacheItem> cache)
    {
        _cache = cache;
    }

    public async Task<BookCacheItem> GetAsync(Guid bookId)
    {
        return await _cache.GetOrAddAsync(
            bookId.ToString(),  // Cache key
            async () => await GetBookFromDatabaseAsync(bookId),  // Factory
            () => new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            }
        );
    }

    private async Task<BookCacheItem> GetBookFromDatabaseAsync(Guid bookId)
    {
        // Fetch from database
    }
}
```

> Use `IDistributedCache<T>` (Redis-backed) for multi-pod deployments. Never use `IMemoryCache` for shared state in K8s.

## Event Bus

### Local Events (Same Process, Same Transaction)

Local events are handled within the same unit of work transaction. Use for intra-module communication:

```csharp
// Event class (in Domain or Application)
public class OrderCreatedEvent
{
    public Order Order { get; set; }
}

// Handler (in Application layer)
public class OrderCreatedEventHandler :
    ILocalEventHandler<OrderCreatedEvent>,
    ITransientDependency
{
    public async Task HandleEventAsync(OrderCreatedEvent eventData)
    {
        // Handle within same transaction
        // If this throws, the entire UoW rolls back
    }
}

// Publish (from entity or application service)
await _localEventBus.PublishAsync(new OrderCreatedEvent { Order = order });
```

### Distributed Events (Cross-Service, Async)

Distributed events are handled asynchronously across services. Use for cross-module/microservice communication:

```csharp
// ETO (Event Transfer Object) — in Domain.Shared
[EventName("MyApp.Order.Created")]
public class OrderCreatedEto
{
    public Guid OrderId { get; set; }
    public string OrderNumber { get; set; }
    public decimal TotalAmount { get; set; }
}

// Handler (in Application layer of consuming service)
public class OrderCreatedEtoHandler :
    IDistributedEventHandler<OrderCreatedEto>,
    ITransientDependency
{
    public async Task HandleEventAsync(OrderCreatedEto eventData)
    {
        // Handle cross-service event
    }
}

// Publish (from entity or application service)
await _distributedEventBus.PublishAsync(
    new OrderCreatedEto { OrderId = order.Id, OrderNumber = order.Number }
);
```

### When to Use Which

| Event Type | Scope | Transaction | Use Case |
|-----------|-------|------------|----------|
| Local | Same process | Same UoW | Side effects within module |
| Distributed | Cross-service | Async/separate | Cross-module, microservice communication |

### Outbox / Inbox Pattern

For reliable distributed events, ABP supports the Outbox/Inbox pattern:
- **Outbox**: Events are stored in DB before publishing (ensures at-least-once delivery)
- **Inbox**: Received events are stored to prevent duplicate processing (idempotency)
- Implement `IHasEventOutbox` and `IHasEventInbox` on your DbContext

## Background Jobs

### Define and Enqueue Jobs

```csharp
// Job arguments
public class EmailSendingArgs
{
    public string EmailAddress { get; set; }
    public string Subject { get; set; }
    public string Body { get; set; }
}

// Job implementation
public class EmailSendingJob : AsyncBackgroundJob<EmailSendingArgs>
{
    public override async Task ExecuteAsync(EmailSendingArgs args)
    {
        // Send email
    }
}

// Enqueue (from application service)
await _backgroundJobManager.EnqueueAsync(new EmailSendingArgs
{
    EmailAddress = "user@example.com",
    Subject = "Welcome",
    Body = "Welcome to our platform!"
});
```

### Background Workers (Periodic)

```csharp
public class MyPeriodicWorker : AsyncPeriodicBackgroundWorkerBase
{
    public MyPeriodicWorker(
        AbpAsyncTimer timer,
        IServiceScopeFactory serviceScopeFactory)
        : base(timer, serviceScopeFactory)
    {
        Timer.Period = 60000; // Run every 60 seconds
    }

    protected override async Task DoWorkAsync(
        PeriodicBackgroundWorkerContext workerContext)
    {
        // Periodic work
    }
}
```

## Distributed Locking

Use `IDistributedLockProvider` for distributed locks in multi-pod K8s deployments:

```csharp
public class OrderService : ITransientDependency
{
    private readonly IDistributedLockProvider _lockProvider;

    public async Task ProcessOrderAsync(Guid orderId)
    {
        await using (var handle = await _lockProvider.TryAcquireAsync(
            $"order-processing-{orderId}",
            TimeSpan.FromSeconds(30)))
        {
            if (handle == null)
            {
                throw new BusinessException("OrderIsBeingProcessed");
            }

            // Process order with exclusive lock
        }
    }
}
```

> Never use `lock(object)` or `SemaphoreSlim` for business logic — they don't work across K8s pods.

## Data Filtering

ABP automatically filters data based on soft-delete and multi-tenancy:

```csharp
// Disable soft-delete filter to include deleted entities
using (DataFilter.Disable<ISoftDelete>())
{
    var allBooks = await _bookRepository.GetListAsync(); // Includes deleted
}

// Disable multi-tenant filter to query across tenants
using (DataFilter.Disable<IMultiTenant>())
{
    var allProducts = await _productRepository.GetListAsync(); // All tenants
}
```

## Entity Cache

Cache frequently accessed entities with automatic invalidation:

```csharp
public class BookCacheItem
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Usage
var cachedBook = await _entityCache.GetAsync<Book, BookCacheItem>(
    bookId,
    cancellationToken
);
```

Entity cache is automatically invalidated when the entity is updated or deleted.

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| `IMemoryCache` for shared state in K8s | Not shared across pods | Use `IDistributedCache<T>` (Redis) |
| `lock(object)` for business logic | Only works within single process | Use `IDistributedLockProvider` |
| Direct `DateTime.Now` in cache expiration | Not testable, ignores timezone | Use `IClock.Now` |
| Hardcoding setting values | Not configurable per tenant/user | Use `ISettingProvider` |
| Publishing distributed events without Outbox | Events can be lost on failure | Use Outbox/Inbox pattern |

## Best Practices Checklist

- Settings defined via `SettingDefinitionProvider`
- Features defined via `FeatureDefinitionProvider`
- `IDistributedCache<T>` used for shared cache (not `IMemoryCache`)
- `IDistributedLockProvider` used for distributed locks (not `lock`)
- Local events for intra-module, distributed events for cross-service
- ETOs defined in `Domain.Shared` for distributed events
- Outbox/Inbox pattern for reliable distributed events
- Background jobs for async/queued work
- `IClock` used for all time-based operations
- Data filters used with `using` blocks for scoped disabling
{
    public string EmailAddress { get; set; }
    public string Subject { get; set; }
    public string Body { get; set; }
}

public class EmailSendingJob : AsyncBackgroundJob<EmailSendingArgs>, ITransientDependency
{
    private readonly IEmailSender _emailSender;

    public EmailSendingJob(IEmailSender emailSender)
    {
        _emailSender = emailSender;
    }

    public override async Task ExecuteAsync(EmailSendingArgs args)
    {
        await _emailSender.SendAsync(args.EmailAddress, args.Subject, args.Body);
    }
}
```

### Enqueue Job
```csharp
await _backgroundJobManager.EnqueueAsync(
    new EmailSendingArgs
    {
        EmailAddress = "user@example.com",
        Subject = "Hello",
        Body = "..."
    },
    delay: TimeSpan.FromMinutes(5) // Optional delay
);
```

## Localization

### Define Resource
```csharp
[LocalizationResourceName("MyModule")]
public class MyModuleResource { }
```

### JSON Structure
```json
{
  "culture": "en",
  "texts": {
    "HelloWorld": "Hello World!",
    "Menu:Books": "Books"
  }
}
```

### Usage
- In `ApplicationService`: Use `L["Key"]` property (already available from base class)
- In other services: Inject `IStringLocalizer<MyResource>`

> **Tip**: ABP base classes already provide commonly used services as properties. Check before injecting:
> - `StringLocalizer` (L), `Clock`, `CurrentUser`, `CurrentTenant`, `GuidGenerator`
> - `AuthorizationService`, `FeatureChecker`, `DataFilter`
> - `LoggerFactory`, `Logger`
> - Methods like `CheckPolicyAsync()` for authorization checks
