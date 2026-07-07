---
name: abp-microservice
description: ABP Microservice solution template - service structure, Integration Services ([IntegrationService]), inter-service HTTP proxies, distributed events with Outbox/Inbox, Entity Cache, RabbitMQ/Redis/YARP setup. Use when working with the ABP microservice solution template or inter-service communication patterns.
---

# ABP Microservice Solution Template

The ABP microservice template provides a complete distributed system with pre-configured infrastructure: RabbitMQ, Redis, YARP API Gateway, OpenIddict authentication, and Docker/K8s deployment.

## Solution Structure

```
MyMicroservice/
├── apps/                           # UI applications
│   ├── web/                        # Admin web application (Blazor/MVC/Angular)
│   ├── public-web/                 # Public website
│   └── auth-server/                # Authentication server (OpenIddict)
├── gateways/                       # BFF pattern — one gateway per UI
│   ├── web-gateway/                # YARP reverse proxy for admin UI
│   └── public-web-gateway/         # YARP reverse proxy for public UI
├── services/                       # Microservices
│   ├── administration/             # Permissions, settings, features, audit logs
│   ├── identity/                   # Users, roles, identity management
│   ├── product/                    # Product catalog (example)
│   └── ordering/                   # Order management (example)
└── etc/
    ├── docker/                     # Docker Compose for local infrastructure
    ├── helm/                       # Helm charts for Kubernetes deployment
    └── abp/                        # ABP Studio metadata
```

## Microservice Structure (NOT Layered!)

Each microservice has a simplified structure — everything in one project (single-layer per service):

```
services/ordering/
├── OrderingService/                # Main project (single-layer)
│   ├── Entities/                   # Domain entities
│   ├── Services/                   # Application services + DTOs
│   ├── IntegrationServices/        # Inter-service communication endpoints
│   ├── Data/                       # DbContext (implements IHasEventInbox, IHasEventOutbox)
│   ├── Migrations/                 # EF Core migrations
│   └── OrderingServiceModule.cs    # Module class
├── OrderingService.Contracts/      # Shared: interfaces, DTOs, ETOs
│   ├── IntegrationServices/        # Integration service interfaces
│   ├── Dtos/                       # Shared DTOs
│   └── EventEtos/                  # Event Transfer Objects
└── OrderingService.Tests/          # Integration tests
```

## Pre-Configured Infrastructure

| Component | Technology | Purpose |
|-----------|-----------|---------|
| API Gateway | YARP | Reverse proxy, routing, auth |
| Authentication | OpenIddict | Centralized auth server |
| Message Broker | RabbitMQ | Distributed events |
| Distributed Cache | Redis | Shared cache across services |
| Database | PostgreSQL (default) | Per-service databases |
| Monitoring | Prometheus + Grafana | Health checks, metrics |
| Orchestration | Docker Compose / K8s | Local dev / production |

## Inter-Service Communication

### 1. Integration Services (Synchronous HTTP)

For synchronous calls between services, use **Integration Services** — NOT regular application services.

#### Complete Setup (7 Steps)

**Step 1: Provider Service — Create Integration Service Interface (Contracts project)**

```csharp
// In CatalogService.Contracts
[IntegrationService]
public interface IProductIntegrationService : IApplicationService
{
    Task<List<ProductDto>> GetProductsByIdsAsync(List<Guid> ids);
}
```

**Step 2: Provider Service — Implement Integration Service**

```csharp
// In CatalogService
[IntegrationService]
public class ProductIntegrationService :
    ApplicationService,
    IProductIntegrationService
{
    private readonly IRepository<Product, Guid> _productRepository;

    public ProductIntegrationService(IRepository<Product, Guid> productRepository)
    {
        _productRepository = productRepository;
    }

    public async Task<List<ProductDto>> GetProductsByIdsAsync(List<Guid> ids)
    {
        var products = await _productRepository.GetListAsync(
            p => ids.Contains(p.Id)
        );
        return ObjectMapper.Map<List<Product>, List<ProductDto>>(products);
    }
}
```

**Step 3: Provider Service — Expose Integration Services**

```csharp
// In CatalogServiceModule.cs
Configure<AbpAspNetCoreMvcOptions>(options =>
{
    options.ExposeIntegrationServices = true;
});
```

**Step 4: Consumer Service — Add Package Reference**

Add reference to provider's `Contracts` project (via ABP Studio or manually).

**Step 5: Consumer Service — Generate Proxies**

```bash
abp generate-proxy -t csharp -u http://localhost:44361 -m catalog --without-contracts
```

**Step 6: Consumer Service — Register HTTP Client Proxies**

```csharp
// In OrderingServiceModule.cs
[DependsOn(typeof(CatalogServiceContractsModule))]
public class OrderingServiceModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddStaticHttpClientProxies(
            typeof(CatalogServiceContractsModule).Assembly,
            "CatalogService"
        );
    }
}
```

**Step 7: Consumer Service — Configure Remote Service URL**

```json
// appsettings.json
{
  "RemoteServices": {
    "CatalogService": {
      "BaseUrl": "http://localhost:44361"
    }
  }
}
```

**Usage:**

```csharp
public class OrderAppService : ApplicationService
{
    private readonly IProductIntegrationService _productIntegrationService;

    public async Task<List<OrderDto>> GetListAsync()
    {
        var orders = await _orderRepository.GetListAsync();
        var productIds = orders.Select(o => o.ProductId).Distinct().ToList();

        // Call remote service via generated proxy
        var products = await _productIntegrationService.GetProductsByIdsAsync(productIds);

        // Combine data...
    }
}
```

> **Why Integration Services?** Application services are designed for UI — they have different authorization, validation, and optimization needs. Integration services are designed specifically for inter-service communication.

### 2. Distributed Events (Asynchronous, RabbitMQ)

Use RabbitMQ-based events for loose coupling between services.

**When to use:**
- Notifying other services about state changes ("order placed", "stock updated")
- Operations that don't need immediate response
- When services should remain independent and decoupled

```csharp
// Define ETO in Contracts project
[EventName("Product.StockChanged")]
public class StockCountChangedEto
{
    public Guid ProductId { get; set; }
    public int NewCount { get; set; }
}

// Publish (in provider service)
await _distributedEventBus.PublishAsync(
    new StockCountChangedEto { ProductId = id, NewCount = newCount }
);

// Subscribe (in consumer service)
public class StockChangedHandler :
    IDistributedEventHandler<StockCountChangedEto>,
    ITransientDependency
{
    public async Task HandleEventAsync(StockCountChangedEto eventData)
    {
        // Handle the event — update local cache, trigger workflow, etc.
    }
}
```

### Outbox/Inbox Pattern

DbContext must implement `IHasEventInbox` and `IHasEventOutbox` for reliable event delivery:

```csharp
public class OrderingServiceDbContext :
    AbpDbContext<OrderingServiceDbContext>,
    IHasEventInbox,   // For receiving events reliably
    IHasEventOutbox   // For publishing events reliably
{
    public DbSet<IncomingEventRecord> IncomingEvents { get; set; }
    public DbSet<OutgoingEventRecord> OutgoingEvents { get; set; }
}
```

## Entity Cache (Performance)

For frequently accessed data from other services, use Entity Cache:

```csharp
// Register in module
context.Services.AddEntityCache<Product, ProductDto, Guid>();

// Use — auto-invalidates on entity changes
public class OrderAppService : ApplicationService
{
    private readonly IEntityCache<ProductDto, Guid> _productCache;

    public async Task<ProductDto> GetProductAsync(Guid id)
    {
        return await _productCache.GetAsync(id);
    }
}
```

## API Gateways (YARP)

Each UI application has its own gateway (BFF pattern):

```json
// gateways/web-gateway/appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "administration": {
        "ClusterId": "administration",
        "Match": { "Path": "/api/administration/{**catch-all}" }
      },
      "identity": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/identity/{**catch-all}" }
      }
    },
    "Clusters": {
      "administration": {
        "Destinations": {
          "administration": { "Address": "http://administration-service/" }
        }
      }
    }
  }
}
```

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| Using Application Services for inter-service calls | Wrong auth, validation, optimization | Use `[IntegrationService]` |
| Direct DB access across services | Tight coupling, no API contract | Use Integration Services or Events |
| Synchronous calls for non-critical operations | Blocks caller, creates dependency chain | Use distributed events |
| Skipping Outbox/Inbox | Events can be lost on failure | Implement `IHasEventInbox`/`IHasEventOutbox` |
| Shared database between services | Tight coupling, no independent scaling | Each service has its own database |

## Best Practices Checklist

- [ ] Each microservice has its own database
- [ ] Integration Services used for synchronous inter-service calls
- [ ] Distributed Events (RabbitMQ) for asynchronous communication
- [ ] Outbox/Inbox pattern for reliable event delivery
- [ ] Entity Cache for frequently accessed cross-service data
- [ ] YARP gateway per UI application (BFF pattern)
- [ ] OpenIddict for centralized authentication
- [ ] Redis for distributed caching
- [ ] Docker Compose for local development
- [ ] Helm charts for Kubernetes deployment
    return await _productCache.GetAsync(id);
}
```

## Pre-Configured Infrastructure

- **RabbitMQ** - Distributed events with Outbox/Inbox
- **Redis** - Distributed cache and locking
- **YARP** - API Gateway
- **OpenIddict** - Auth server

## Best Practices

- **Choose communication wisely** - Synchronous for queries needing immediate data, asynchronous for notifications and state changes
- **Use Integration Services** - Not application services for inter-service calls
- **Cache remote data** - Use Entity Cache or IDistributedCache for frequently accessed data
- **Share only Contracts** - Never share implementations
- **Idempotent handlers** - Events may be delivered multiple times
- **Database per service** - Each service owns its database
