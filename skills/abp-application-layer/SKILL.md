---
name: abp-application-layer
description: ABP Application Services, DTOs, CRUD service, object mapping (Mapperly/AutoMapper), validation, error handling. Use when creating or reviewing application services, DTOs, or working in the Application or Application.Contracts projects.
user-invocable: false
---

# ABP Application Layer

ABP application services implement use cases by orchestrating domain objects. They act as a **facade** between the presentation layer and the domain layer. Application services should NOT contain business logic — that belongs in domain entities and domain services.

## Application Service Structure

### Interface (Application.Contracts)

Interfaces must inherit from `IApplicationService`. Method naming follows conventions:

```csharp
public interface IBookAppService : IApplicationService
{
    Task<BookDto> GetAsync(Guid id);
    Task<PagedResultDto<BookListItemDto>> GetListAsync(GetBookListInput input);
    Task<BookDto> CreateAsync(CreateBookDto input);
    Task<BookDto> UpdateAsync(Guid id, UpdateBookDto input);
    Task DeleteAsync(Guid id);
}
```

### Method Naming Conventions

| Operation | Method Name | Parameters |
|-----------|------------|------------|
| Get single | `GetAsync` | `Guid id` |
| Get list | `GetListAsync` | `GetXxxListInput input` |
| Create | `CreateAsync` | `CreateXxxDto input` |
| Update | `UpdateAsync` | `Guid id, UpdateXxxDto input` |
| Delete | `DeleteAsync` | `Guid id` |

> Do NOT repeat entity name in method names: use `GetAsync` not `GetBookAsync`. The entity context is already in the interface name `IBookAppService`.

> ID is passed as a **separate parameter**, NOT inside the UpdateDto.

### Implementation (Application)

Inherit from `ApplicationService`. For reusable modules, make public methods `virtual` for extensibility; for your own application, `virtual` is optional:

```csharp
public class BookAppService : ApplicationService, IBookAppService
{
    private readonly IBookRepository _bookRepository;
    private readonly BookManager _bookManager;

    public BookAppService(
        IBookRepository bookRepository,
        BookManager bookManager)
    {
        _bookRepository = bookRepository;
        _bookManager = bookManager;
    }

    public virtual async Task<BookDto> GetAsync(Guid id)
    {
        var book = await _bookRepository.GetAsync(id);
        return ObjectMapper.Map<Book, BookDto>(book);
    }

    [Authorize(BookStorePermissions.Books.Create)]
    public virtual async Task<BookDto> CreateAsync(CreateBookDto input)
    {
        var book = await _bookManager.CreateAsync(
            GuidGenerator.Create(),
            input.Name,
            input.Price
        );
        await _bookRepository.InsertAsync(book);
        return ObjectMapper.Map<Book, BookDto>(book);
    }

    [Authorize(BookStorePermissions.Books.Edit)]
    public virtual async Task<BookDto> UpdateAsync(Guid id, UpdateBookDto input)
    {
        var book = await _bookRepository.GetAsync(id);
        await _bookManager.ChangeNameAsync(book, input.Name);
        book.SetPrice(input.Price);
        await _bookRepository.UpdateAsync(book);
        return ObjectMapper.Map<Book, BookDto>(book);
    }

    [Authorize(BookStorePermissions.Books.Delete)]
    public virtual async Task DeleteAsync(Guid id)
    {
        await _bookRepository.DeleteAsync(id);
    }
}
```

### ApplicationService Base Properties

When you inherit from `ApplicationService`, these are pre-injected — no need to inject them manually:

| Property | Type | Description |
|----------|------|-------------|
| `GuidGenerator` | `IGuidGenerator` | Generate sequential GUIDs |
| `Clock` | `IClock` | Current time (use instead of `DateTime`) |
| `CurrentUser` | `ICurrentUser` | Authenticated user info |
| `CurrentTenant` | `ICurrentTenant` | Multi-tenancy context |
| `L` | `IStringLocalizer` | Localization |
| `AuthorizationService` | `IAuthorizationService` | Permission checks |
| `FeatureChecker` | `IFeatureChecker` | Feature availability |
| `DataFilter` | `IDataFilter` | Soft-delete/tenant filter control |
| `UnitOfWorkManager` | `IUnitOfWorkManager` | Unit of work management |
| `LoggerFactory` | `ILoggerFactory` | Create loggers |
| `ObjectMapper` | `IObjectMapper` | Object-to-object mapping |
| `LazyServiceProvider` | `ILazyServiceProvider` | Lazy service resolution |

### Data Access Pattern

1. **Load** entities from repositories
2. **Mutate** using domain methods (not direct property setters)
3. **Save** by calling `UpdateAsync` explicitly (do NOT assume change tracking)

```csharp
public virtual async Task<BookDto> UpdateAsync(Guid id, UpdateBookDto input)
{
    // 1. Load
    var book = await _bookRepository.GetAsync(id);

    // 2. Mutate via domain methods
    book.ChangeName(input.Name);
    book.SetPrice(input.Price);

    // 3. Save explicitly
    await _bookRepository.UpdateAsync(book);

    return ObjectMapper.Map<Book, BookDto>(book);
}
```

## DTO Conventions

### DTO Location

Define DTOs in `*.Application.Contracts` project. This allows sharing with clients (Blazor, HttpApi.Client, Angular proxy generation).

### DTO Naming Conventions

| Purpose | Convention | Example |
|---------|------------|---------|
| Query input | `Get{Entity}Input` | `GetBookInput` |
| List query input | `Get{Entity}ListInput` | `GetBookListInput` |
| Create input | `Create{Entity}Dto` | `CreateBookDto` |
| Update input | `Update{Entity}Dto` | `UpdateBookDto` |
| Single entity output | `{Entity}Dto` | `BookDto` |
| List item output | `{Entity}ListItemDto` | `BookListItemDto` |

### Input DTO Rules

- Do NOT include unused properties
- Do NOT share input DTOs between methods (each method gets its own)
- Do NOT use inheritance between input DTOs (except rare abstract base DTO cases)
- Do NOT put ID inside UpdateDto — pass as separate parameter
- Use data annotations for validation
- Reuse constants from `Domain.Shared` for lengths/ranges

### Output DTO Strategy

- Prefer a **Basic DTO** and a **Detailed DTO**; avoid many variants
- Detailed DTOs: include reference details as nested basic DTOs
- Avoid duplicating raw FK IDs unnecessarily when reference details are included

```csharp
// Basic DTO (for list items)
public class BookListItemDto : EntityDto<Guid>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Detailed DTO (for single entity view)
public class BookDto : AuditedEntityDto<Guid>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string Description { get; set; }
    public AuthorDto Author { get; set; }  // Nested basic DTO
}
```

## Validation

### Data Annotations (Primary Approach)

```csharp
public class CreateBookDto
{
    [Required]
    [StringLength(128, MinimumLength = 3)]
    public string Name { get; set; }

    [Required]
    [Range(0.01, 9999.99)]
    public decimal Price { get; set; }

    [DataType(DataType.Date)]
    public DateTime PublishDate { get; set; }
}
```

### IValidatableObject (Cross-Field Validation)

Only use for application-level validation that can't be expressed with data annotations:

```csharp
public class CreateBookDto : IValidatableObject
{
    public string Name { get; set; }
    public string Description { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (Name == Description)
        {
            yield return new ValidationResult(
                "Name and Description cannot be the same!",
                new[] { nameof(Name), nameof(Description) }
            );
        }
    }
}
```

> **Domain rules vs Application rules**: Domain rules (business invariants) belong in entity constructors/domain services. Application rules (input format, required fields) belong in DTO validation.

### FluentValidation

```csharp
public class CreateBookDtoValidator : AbstractValidator<CreateBookDto>
{
    public CreateBookDtoValidator()
    {
        RuleFor(x => x.Name).NotEmpty().Length(3, 128);
        RuleFor(x => x.Price).GreaterThan(0);
    }
}
```

## Error Handling

### BusinessException (Domain Rule Violations)

Use namespaced error codes for localization:

```csharp
throw new BusinessException("BookStore:BookNameAlreadyExists")
    .WithData("Name", name);
```

Configure localization mapping in module:

```csharp
Configure<AbpExceptionLocalizationOptions>(options =>
{
    options.MapCodeNamespace("BookStore", typeof(BookStoreResource));
});
```

### EntityNotFoundException

```csharp
var book = await _bookRepository.FindAsync(id);
if (book == null)
{
    throw new EntityNotFoundException(typeof(Book), id);
}
```

### UserFriendlyException

For user-facing error messages:

```csharp
throw new UserFriendlyException(L["BookNotAvailable"]);
```

### HTTP Status Code Mapping

ABP automatically maps exceptions to HTTP status codes. This mapping is **configurable**:

| Exception | Typical HTTP Status |
|-----------|-------------------|
| `AbpValidationException` | 400 Bad Request |
| `AbpAuthorizationException` | 401/403 |
| `EntityNotFoundException` | 404 Not Found |
| `BusinessException` | 403 (configurable) |
| Other exceptions | 500 Internal Server Error |

## Auto API Controllers

ABP automatically generates HTTP API controllers for application services:

- Interface must inherit `IApplicationService` (which has `[RemoteService]` attribute)
- HTTP methods determined by method name prefix:
  - `Get`/`GetList` → `GET`
  - `Create`/`CreateAsync` → `POST`
  - `Update`/`UpdateAsync` → `PUT`
  - `Delete`/`DeleteAsync` → `DELETE`
- Use `[RemoteService(false)]` to disable auto API for specific methods
- Use `[RemoteService(true)]` to explicitly enable

## Object Mapping

ABP supports both **Mapperly** (compile-time, default) and **AutoMapper** (runtime). Check the project's active mapping library first.

### Mapperly (Compile-Time — Default)

Define mappers as partial classes with `[Mapper]` attribute:

```csharp
[Mapper]
public partial class BookMapper
{
    public partial BookDto MapToDto(Book book);
    public partial List<BookDto> MapToDtoList(List<Book> books);
    public partial void MapToEntity(UpdateBookDto input, Book book);
}
```

Register in module:

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddSingleton<BookMapper>();
}
```

Usage:

```csharp
public class BookAppService : ApplicationService
{
    private readonly BookMapper _bookMapper;

    public BookAppService(BookMapper bookMapper)
    {
        _bookMapper = bookMapper;
    }

    public BookDto GetBook(Book book)
    {
        return _bookMapper.MapToDto(book);
    }
}
```

> Mapperly generates mapping code at compile-time, providing better performance than runtime mappers.

### AutoMapper (Runtime)

If the solution uses AutoMapper, mappings are defined in `Profile` classes:

```csharp
public class BookStoreApplicationAutoMapperProfile : Profile
{
    public BookStoreApplicationAutoMapperProfile()
    {
        CreateMap<Book, BookDto>();
        CreateMap<CreateBookDto, Book>();
        CreateMap<UpdateBookDto, Book>();
    }
}
```

Usage via `ObjectMapper` property (available in `ApplicationService` base):

```csharp
var dto = ObjectMapper.Map<Book, BookDto>(book);
```

## Cross-Service Calls

- Do NOT call other application services **within the same module**
- For reuse, push logic into domain layer or extract shared helpers
- You MAY call other modules' application services via their `Application.Contracts` interfaces

```csharp
// ✅ OK: Calling another module's service via contracts
public class MyAppService : ApplicationService
{
    private readonly IIdentityUserAppService _identityUserAppService;
    // IIdentityUserAppService is from Volo.Abp.Identity.Application.Contracts
}

// ❌ Wrong: Calling another service in the same module
public class BookAppService : ApplicationService
{
    private readonly IAuthorAppService _authorAppService; // Don't do this!
}
```

## File Handling

Do NOT use web types like `IFormFile` or `Stream` in application services. Controllers handle upload; pass `byte[]` to application services:

```csharp
// ✅ Correct: Application service accepts byte[]
public interface IFileAppService : IApplicationService
{
    Task UploadAsync(byte[] fileContent, string fileName);
}

// ❌ Wrong: Application service using web types
public interface IFileAppService : IApplicationService
{
    Task UploadAsync(IFormFile file); // Don't do this!
}
```

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| Entity name in method (`GetBookAsync`) | Redundant — context is in interface name | Use `GetAsync` |
| ID inside UpdateDto | Confuses resource identity with update data | Pass `id` as separate parameter |
| Calling other app services in same module | Creates circular dependencies, violates layering | Push logic to domain layer |
| `IFormFile`/`Stream` in app service | Couples application to web layer | Accept `byte[]` from controllers |
| Business logic in app service | Violates DDD — app services are coordinators | Put in domain entities/services |
| Not calling `UpdateAsync` after mutation | Relies on change tracking (not guaranteed) | Always call `UpdateAsync` explicitly |
| Sharing input DTOs between methods | Couples different use cases | Each method gets its own DTO |
| Inheritance between input DTOs | Creates fragile hierarchies | Keep DTOs flat and independent |

## Best Practices Checklist

- Interface inherits `IApplicationService`
- Implementation inherits `ApplicationService`
- Public methods are `virtual` (required for reusable modules, optional for your own app)
- Method names follow conventions (`GetAsync`, `CreateAsync`, etc.)
- ID passed as separate parameter, not inside UpdateDto
- DTOs in `Application.Contracts` project
- Each method has its own input DTO
- Data annotations used for validation
- Domain rules in entities/services, not in DTO validation
- `BusinessException` with namespaced error codes
- `UpdateAsync` called explicitly after mutations
- No cross-service calls within same module
- No web types (`IFormFile`, `Stream`) in app services
- Use base class properties (`Clock`, `GuidGenerator`, `L`) instead of injecting
