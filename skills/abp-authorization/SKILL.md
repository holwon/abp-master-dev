---
name: abp-authorization
description: ABP permission system - PermissionDefinitionProvider, [Authorize] attribute, CheckPolicyAsync, IsGrantedAsync, ICurrentUser, IPermissionManager, multi-tenancy side. Use when working with permissions, authorization, role-based access, or security in ABP projects.
---

# ABP Authorization

ABP provides a comprehensive permission-based authorization system. Permissions are defined in `Application.Contracts` and can be checked declaratively or programmatically.

## Permission Definition

### Permission Constants (Application.Contracts)

Define permission names as constants for type safety:

```csharp
public static class BookStorePermissions
{
    public const string GroupName = "BookStore";

    public static class Books
    {
        public const string Default = GroupName + ".Books";
        public const string Create = Default + ".Create";
        public const string Edit = Default + ".Edit";
        public const string Delete = Default + ".Delete";
    }
}
```

### Permission Definition Provider (Application.Contracts)

```csharp
public class BookStorePermissionDefinitionProvider : PermissionDefinitionProvider
{
    public override void Define(IPermissionDefinitionContext context)
    {
        var bookStoreGroup = context.AddGroup(
            BookStorePermissions.GroupName,
            L("Permission:BookStore"));

        var booksPermission = bookStoreGroup.AddPermission(
            BookStorePermissions.Books.Default,
            L("Permission:Books"));

        booksPermission.AddChild(
            BookStorePermissions.Books.Create,
            L("Permission:Books.Create"));

        booksPermission.AddChild(
            BookStorePermissions.Books.Edit,
            L("Permission:Books.Edit"));

        booksPermission.AddChild(
            BookStorePermissions.Books.Delete,
            L("Permission:Books.Delete"));
    }

    private static LocalizableString L(string name)
    {
        return LocalizableString.Create<BookStoreResource>(name);
    }
}
```

### Permission Naming Convention

`ModuleName.Permission` (e.g., `BookStore.Books.Create`, `Identity.Users.Create`). Use hierarchical structure with dots.

## Using Permissions

### Declarative (Attribute)

```csharp
public class BookAppService : ApplicationService
{
    [Authorize(BookStorePermissions.Books.Create)]
    public virtual async Task<BookDto> CreateAsync(CreateBookDto input)
    {
        // Only users with Books.Create permission can execute
    }

    [Authorize(BookStorePermissions.Books.Default)]
    public virtual async Task<BookDto> GetAsync(Guid id)
    {
        // Only users with Books permission can execute
    }
}
```

### Programmatic Check

```csharp
public class BookAppService : ApplicationService
{
    public async Task DoSomethingAsync()
    {
        // Check and throw AbpAuthorizationException if not granted
        await CheckPolicyAsync(BookStorePermissions.Books.Edit);

        // Check without throwing — returns boolean
        if (await IsGrantedAsync(BookStorePermissions.Books.Delete))
        {
            // Has permission — show delete button, etc.
        }
    }
}
```

> `CheckPolicyAsync()` and `IsGrantedAsync()` are available in `ApplicationService` and `AbpController` base classes.

### Allow Anonymous Access

```csharp
[AllowAnonymous]
public virtual async Task<BookDto> GetPublicBookAsync(Guid id)
{
    // No authentication required
}
```

## Current User

Access authenticated user info via `CurrentUser` property (available in `ApplicationService`, `DomainService`, `AbpController`):

```csharp
public class BookAppService : ApplicationService
{
    public async Task DoSomethingAsync()
    {
        // CurrentUser is available from base class — no injection needed
        Guid? userId = CurrentUser.Id;           // Null if not authenticated
        string userName = CurrentUser.UserName;
        string email = CurrentUser.Email;
        bool isAuthenticated = CurrentUser.IsAuthenticated;
        string[] roles = CurrentUser.Roles;
        Guid? tenantId = CurrentUser.TenantId;

        // Check specific role
        bool isAdmin = CurrentUser.IsInRole("admin");
    }
}

// In other services, inject ICurrentUser
public class MyService : ITransientDependency
{
    private readonly ICurrentUser _currentUser;

    public MyService(ICurrentUser currentUser)
    {
        _currentUser = currentUser;
    }
}
```

### Ownership Validation

```csharp
public async Task UpdateMyBookAsync(Guid bookId, UpdateBookDto input)
{
    var book = await _bookRepository.GetAsync(bookId);

    // Validate ownership
    if (book.CreatorId != CurrentUser.Id)
    {
        throw new AbpAuthorizationException("You can only edit your own books");
    }

    // Update book...
}
```

## Resource-Based Authorization

For scenarios where authorization depends on a specific entity instance:

```csharp
public class BookAppService : ApplicationService
{
    private readonly IAuthorizationService _authorizationService;

    public async Task DeleteAsync(Guid id)
    {
        var book = await _bookRepository.GetAsync(id);

        // Check if current user is the creator
        if (book.CreatorId != CurrentUser.Id)
        {
            throw new AbpAuthorizationException();
        }

        await _bookRepository.DeleteAsync(book);
    }
}
```

## Multi-Tenancy Permissions

Control permission availability per tenant side:

```csharp
bookStoreGroup.AddPermission(
    BookStorePermissions.Books.Default,
    L("Permission:Books"),
    multiTenancySide: MultiTenancySides.Tenant  // Only for tenants
);

// Options:
// MultiTenancySides.Host   — Only host users
// MultiTenancySides.Tenant — Only tenant users
// MultiTenancySides.Both   — Both host and tenant (default)
```

## Feature-Dependent Permissions

Make permissions depend on features:

```csharp
booksPermission.RequireFeatures("BookStore.PremiumFeature");
```

## Permission Management (Programmatic)

Grant or revoke permissions programmatically:

```csharp
public class MyService : ITransientDependency
{
    private readonly IPermissionManager _permissionManager;

    public MyService(IPermissionManager permissionManager)
    {
        _permissionManager = permissionManager;
    }

    public async Task GrantToUserAsync(Guid userId, string permissionName)
    {
        await _permissionManager.SetForUserAsync(userId, permissionName, true);
    }

    public async Task RevokeFromUserAsync(Guid userId, string permissionName)
    {
        await _permissionManager.SetForUserAsync(userId, permissionName, false);
    }

    public async Task GrantToRoleAsync(string roleName, string permissionName)
    {
        await _permissionManager.SetForRoleAsync(roleName, permissionName, true);
    }
}
```

## Security Best Practices

- Never trust client input for user identity — always use `CurrentUser` from server
- Use `CurrentUser` property from base class or inject `ICurrentUser`
- Validate ownership in application service methods for resource-level access
- Use `CheckPolicyAsync()` for mandatory checks (throws on failure)
- Use `IsGrantedAsync()` for conditional UI logic
- Always authorize both the endpoint AND the service method
- Permission names should be constants, never hardcoded strings

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| Hardcoding permission strings | Typo-prone, no refactoring support | Use constants in `*Permissions` class |
| Trusting client-side auth only | Client can be bypassed | Always check on server |
| Not checking ownership | Users can access others' data | Validate `CreatorId == CurrentUser.Id` |
| Using `CurrentUser` without null check | `CurrentUser.Id` is null for anonymous | Check `IsAuthenticated` first |
| Skipping permission on service method | Auto API exposes it without auth | Add `[Authorize]` on service methods |

## Best Practices Checklist

- [ ] Permission constants defined in `Application.Contracts`
- [ ] `PermissionDefinitionProvider` registered in `Application.Contracts` module
- [ ] Hierarchical permission structure (parent → child)
- [ ] `[Authorize]` on all protected application service methods
- [ ] `CheckPolicyAsync()` for mandatory checks
- [ ] `IsGrantedAsync()` for conditional logic
- [ ] Ownership validation for resource-level access
- [ ] `CurrentUser` used for identity (never from client input)
- [ ] Multi-tenancy side specified when needed
- [ ] Feature-dependent permissions when applicable
- Filter queries by current user when appropriate
- Don't expose sensitive fields in DTOs
