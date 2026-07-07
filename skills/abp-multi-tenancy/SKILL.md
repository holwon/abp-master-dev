---
name: abp-multi-tenancy
description: ABP Multi-Tenancy - IMultiTenant interface, CurrentTenant, CurrentTenant.Change(), DataFilter.Disable(IMultiTenant), tenant resolution order, database-per-tenant. Use when working with multi-tenant features, tenant-specific data isolation, or switching tenant context.
---

# ABP Multi-Tenancy

ABP provides built-in multi-tenancy with automatic data filtering, tenant resolution, and database isolation options. Multi-tenant entities are automatically scoped to the current tenant.

## Making Entities Multi-Tenant

Implement `IMultiTenant` interface to make entities tenant-aware:

```csharp
public class Product : AggregateRoot<Guid>, IMultiTenant
{
    public Guid? TenantId { get; set; } // Required by IMultiTenant

    public string Name { get; private set; }
    public decimal Price { get; private set; }

    protected Product() { }

    public Product(Guid id, string name, decimal price) : base(id)
    {
        Name = name;
        Price = price;
        // TenantId is automatically set from CurrentTenant.Id by ABP
    }
}
```

**Key points:**
- `TenantId` is **nullable** — `null` means entity belongs to Host side
- ABP **automatically filters** queries by current tenant (via `DataFilter`)
- ABP **automatically sets** `TenantId` when creating new entities
- You do NOT need to manually set `TenantId` in most cases

## Accessing Current Tenant

Use `CurrentTenant` property (available in all base classes) or inject `ICurrentTenant`:

```csharp
public class ProductAppService : ApplicationService
{
    public async Task DoSomethingAsync()
    {
        // Available from base class — no injection needed
        Guid? tenantId = CurrentTenant.Id;        // null for host
        string tenantName = CurrentTenant.Name;    // null for host
        bool isAvailable = CurrentTenant.IsAvailable; // true if tenant context exists
    }
}

// In other services, inject ICurrentTenant
public class MyService : ITransientDependency
{
    private readonly ICurrentTenant _currentTenant;

    public MyService(ICurrentTenant currentTenant)
    {
        _currentTenant = currentTenant;
    }
}
```

## Switching Tenant Context

Use `CurrentTenant.Change()` to temporarily switch tenant. **Always use with `using` statement:**

```csharp
public class ProductManager : DomainService
{
    private readonly IRepository<Product, Guid> _productRepository;

    public async Task<long> GetProductCountForTenantAsync(Guid? tenantId)
    {
        // Switch to specific tenant context
        using (CurrentTenant.Change(tenantId))
        {
            return await _productRepository.GetCountAsync();
        }
        // Automatically restored to previous tenant after using block
    }

    public async Task DoHostOperationAsync()
    {
        // Switch to host context (null = host)
        using (CurrentTenant.Change(null))
        {
            // Operations here are in host context
            var allTenants = await _tenantRepository.GetListAsync();
        }
    }
}
```

> **Critical**: Always use `Change()` with a `using` statement. Nested `Change()` calls are supported — the original context is restored when the inner scope exits.

## Disabling Multi-Tenant Filter

To query across all tenants (only works with single/shared database):

```csharp
public class ProductManager : DomainService
{
    private readonly IRepository<Product, Guid> _productRepository;

    public async Task<long> GetTotalProductCountAsync()
    {
        // DataFilter is available from base class
        using (DataFilter.Disable<IMultiTenant>())
        {
            return await _productRepository.GetCountAsync();
            // Returns count from ALL tenants + host
        }
    }
}
```

> This does NOT work with database-per-tenant architecture — each tenant has its own physical database.

## Database Architecture Options

| Approach | Description | Use Case | Connection String |
|----------|-------------|----------|-------------------|
| **Single Database** | All tenants share one database | Simple, cost-effective, small tenants | One shared connection string |
| **Database per Tenant** | Each tenant has dedicated database | Data isolation, compliance, large tenants | Per-tenant connection string |
| **Hybrid** | Mix of shared and dedicated | Flexible, premium tenants get dedicated DB | Mix of both |

Connection strings are configured per tenant in the Tenant Management module.

## Enabling Multi-Tenancy

```csharp
Configure<AbpMultiTenancyOptions>(options =>
{
    options.IsEnabled = true; // Enabled by default in ABP templates
});
```

Check `MultiTenancyConsts.IsEnabled` in your solution for centralized control.

## Tenant Resolution

ABP resolves the current tenant from these sources (in priority order):

1. **Current user's claims** — From authentication token (`tenantid` claim)
2. **Query string** — `?__tenant=tenant-name`
3. **Route value** — `/{__tenant}/...`
4. **Request header** — `__tenant` header
5. **Cookie** — `__tenant` cookie
6. **Domain / Subdomain** — e.g., `tenant1.myapp.com`

### Subdomain-Based Resolution

```csharp
Configure<AbpTenantResolveOptions>(options =>
{
    options.AddDomainTenantResolver("{0}.myapp.com");
    // tenant1.myapp.com → resolves to tenant "tenant1"
});
```

## Best Practices

1. **Always implement `IMultiTenant`** for tenant-specific entities
2. **Never manually filter by `TenantId`** — ABP does it automatically via `DataFilter`
3. **Don't change `TenantId` after creation** — it effectively moves entity between tenants
4. **Use `Change()` scope carefully** — nested scopes are supported, always use `using`
5. **Test both host and tenant contexts** — ensure proper data isolation
6. **Consider nullable `TenantId`** — `null` means host-level entity, accessible to all
7. **Be aware of database architecture** — `DataFilter.Disable<IMultiTenant>()` doesn't work with DB-per-tenant

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| Manually filtering by `TenantId` in queries | ABP already does this automatically | Trust the `DataFilter` |
| Changing `TenantId` after entity creation | Moves entity between tenants, breaks audit | Never change `TenantId` |
| `CurrentTenant.Change()` without `using` | Tenant context leaks | Always use `using` statement |
| Assuming `CurrentTenant.Id` is always non-null | Host context has null tenant | Check `CurrentTenant.IsAvailable` |
| `DataFilter.Disable<IMultiTenant>()` with DB-per-tenant | Can't query across physical databases | Use host-level aggregation |

## Best Practices Checklist

- Tenant-specific entities implement `IMultiTenant`
- `TenantId` is nullable (`Guid?`)
- No manual `TenantId` filtering in queries
- `CurrentTenant.Change()` always used with `using`
- Both host and tenant contexts tested
- Database architecture chosen consciously (single vs per-tenant vs hybrid)
- Tenant resolution order understood
- `MultiTenancyConsts.IsEnabled` used for centralized control
4. HTTP header (`__tenant`)
5. Cookie (`__tenant`)
6. Domain/subdomain (if configured)

For subdomain-based resolution:
```csharp
Configure<AbpTenantResolveOptions>(options =>
{
    options.AddDomainTenantResolver("{0}.mydomain.com");
});
```
