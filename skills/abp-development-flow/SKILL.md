---
name: abp-development-flow
description: ABP development workflow - step-by-step guide for adding new entities, migrations, application services, localization, permissions, and tests. Use when adding a new feature, entity, or creating a full CRUD flow in an ABP project.
user-invocable: false
---

# ABP Development Workflow

Step-by-step guide for adding a new entity with full DDD patterns in an ABP layered application.

## Adding a New Entity (Complete 11-Step Flow)

### Step 1: Domain Entity

Create the entity in `*.Domain/Entities/` (layered) or `Entities/` (single-layer/microservice):

```csharp
public class Book : AggregateRoot<Guid>
{
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    public Guid AuthorId { get; private set; }
    public DateTime PublishDate { get; private set; }

    protected Book() { }  // Required for ORM

    public Book(Guid id, string name, decimal price, Guid authorId, DateTime publishDate)
        : base(id)
    {
        Name = Check.NotNullOrWhiteSpace(name, nameof(name));
        SetPrice(price);
        AuthorId = authorId;
        PublishDate = publishDate;
    }

    public void SetPrice(decimal price)
    {
        Price = Check.Range(price, nameof(price), 0, 9999);
    }

    public void ChangeName(string newName)
    {
        Name = Check.NotNullOrWhiteSpace(newName, nameof(newName));
    }
}
```

### Step 2: Domain.Shared Constants & Enums

Add constants and enums in `*.Domain.Shared/`:

```csharp
public static class BookConsts
{
    public const int MaxNameLength = 128;
    public const int MaxDescriptionLength = 2000;
}

public enum BookType
{
    Novel,
    Science,
    Biography,
    History
}
```

### Step 3: Repository Interface (Optional)

Define custom repository in `*.Domain/` only if you need custom query methods. For simple CRUD, skip this step and use generic `IRepository<Book, Guid>` directly:

```csharp
// Only create if custom queries are needed
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

### Step 4: EF Core Configuration

In `*.EntityFrameworkCore/`:

**Add DbSet to DbContext:**

```csharp
public DbSet<Book> Books { get; set; }
```

**Add entity configuration in extension method:**

```csharp
builder.Entity<Book>(b =>
{
    b.ToTable(MyProjectConsts.DbTablePrefix + "Books", MyProjectConsts.DbSchema);
    b.ConfigureByConvention();  // MUST call first

    b.Property(x => x.Name)
        .IsRequired()
        .HasMaxLength(BookConsts.MaxNameLength);

    b.Property(x => x.Price)
        .HasColumnType("decimal(18,2)");

    b.HasIndex(x => x.Name);

    b.HasOne<Author>()
        .WithMany()
        .HasForeignKey(x => x.AuthorId)
        .OnDelete(DeleteBehavior.Restrict);
});
```

**Repository Implementation (only if custom interface defined in Step 3):**

```csharp
public class BookRepository :
    EfCoreRepository<MyProjectDbContext, Book, Guid>,
    IBookRepository
{
    public BookRepository(IDbContextProvider<MyProjectDbContext> dbContextProvider)
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

### Step 5: Run Migration

```bash
cd src/MyProject.EntityFrameworkCore
dotnet ef migrations add AddedBookEntity
dotnet run --project ../MyProject.DbMigrator
```

### Step 6: Application.Contracts — DTOs & Service Interface

```csharp
// Output DTO
public class BookDto : AuditedEntityDto<Guid>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public Guid AuthorId { get; set; }
    public string AuthorName { get; set; }  // From navigation
    public DateTime PublishDate { get; set; }
}

// Create input DTO
public class CreateBookDto
{
    [Required]
    [StringLength(BookConsts.MaxNameLength)]
    public string Name { get; set; }

    [Required]
    [Range(0, 9999)]
    public decimal Price { get; set; }

    [Required]
    public Guid AuthorId { get; set; }

    [DataType(DataType.Date)]
    public DateTime PublishDate { get; set; }
}

// Update input DTO
public class UpdateBookDto
{
    [Required]
    [StringLength(BookConsts.MaxNameLength)]
    public string Name { get; set; }

    [Required]
    [Range(0, 9999)]
    public decimal Price { get; set; }
}

// Service interface
public interface IBookAppService : IApplicationService
{
    Task<BookDto> GetAsync(Guid id);
    Task<PagedResultDto<BookDto>> GetListAsync(PagedAndSortedResultRequestDto input);
    Task<BookDto> CreateAsync(CreateBookDto input);
    Task<BookDto> UpdateAsync(Guid id, UpdateBookDto input);
    Task DeleteAsync(Guid id);
}
```

### Step 7: Object Mapping

**Mapperly (default):**

```csharp
[Mapper]
public partial class BookMapper
{
    public partial BookDto MapToDto(Book book);
    public partial List<BookDto> MapToDtoList(List<Book> books);
}
```

Register in module:

```csharp
context.Services.AddSingleton<BookMapper>();
```

### Step 8: Application Service Implementation

```csharp
public class BookAppService : ApplicationService, IBookAppService
{
    private readonly IRepository<Book, Guid> _bookRepository; // Or IBookRepository
    private readonly BookMapper _bookMapper;

    public BookAppService(
        IRepository<Book, Guid> bookRepository,
        BookMapper bookMapper)
    {
        _bookRepository = bookRepository;
        _bookMapper = bookMapper;
    }

    public async Task<BookDto> GetAsync(Guid id)
    {
        var book = await _bookRepository.GetAsync(id);
        return _bookMapper.MapToDto(book);
    }

    public async Task<PagedResultDto<BookDto>> GetListAsync(
        PagedAndSortedResultRequestDto input)
    {
        var queryable = await _bookRepository.GetQueryableAsync();
        var totalCount = await queryable.CountAsync();
        var books = await queryable
            .OrderBy(b => b.Name)
            .PageBy(input)
            .ToListAsync();

        return new PagedResultDto<BookDto>(
            totalCount,
            _bookMapper.MapToDtoList(books)
        );
    }

    [Authorize(MyProjectPermissions.Books.Create)]
    public async Task<BookDto> CreateAsync(CreateBookDto input)
    {
        var book = new Book(
            GuidGenerator.Create(),
            input.Name,
            input.Price,
            input.AuthorId,
            input.PublishDate
        );

        await _bookRepository.InsertAsync(book);
        return _bookMapper.MapToDto(book);
    }

    [Authorize(MyProjectPermissions.Books.Edit)]
    public async Task<BookDto> UpdateAsync(Guid id, UpdateBookDto input)
    {
        var book = await _bookRepository.GetAsync(id);
        book.ChangeName(input.Name);
        book.SetPrice(input.Price);
        await _bookRepository.UpdateAsync(book);
        return _bookMapper.MapToDto(book);
    }

    [Authorize(MyProjectPermissions.Books.Delete)]
    public async Task DeleteAsync(Guid id)
    {
        await _bookRepository.DeleteAsync(id);
    }
}
```

### Step 9: Localization

In `*.Domain.Shared/Localization/{ResourceName}/en.json`:

```json
{
  "Book": "Book",
  "Books": "Books",
  "BookName": "Name",
  "BookPrice": "Price",
  "NewBook": "New Book",
  "EditBook": "Edit Book",
  "BookDeletionConfirmationMessage": "Are you sure to delete the book {0}?"
}
```

### Step 10: Permissions

```csharp
public static class MyProjectPermissions
{
    public const string GroupName = "MyProject";

    public static class Books
    {
        public const string Default = GroupName + ".Books";
        public const string Create = Default + ".Create";
        public const string Edit = Default + ".Edit";
        public const string Delete = Default + ".Delete";
    }
}
```

Register in `PermissionDefinitionProvider`:

```csharp
var booksPermission = myGroup.AddPermission(
    MyProjectPermissions.Books.Default,
    L("Permission:Books"));

booksPermission.AddChild(MyProjectPermissions.Books.Create, L("Permission:Books.Create"));
booksPermission.AddChild(MyProjectPermissions.Books.Edit, L("Permission:Books.Edit"));
booksPermission.AddChild(MyProjectPermissions.Books.Delete, L("Permission:Books.Delete"));
```

### Step 11: Tests

```csharp
public class BookAppService_Tests : MyProjectApplicationTestBase
{
    private readonly IBookAppService _bookAppService;

    public BookAppService_Tests()
    {
        _bookAppService = GetRequiredService<IBookAppService>();
    }

    [Fact]
    public async Task Should_Create_Book()
    {
        var result = await _bookAppService.CreateAsync(new CreateBookDto
        {
            Name = "Test Book",
            Price = 19.99m,
            AuthorId = Guid.NewGuid(),
            PublishDate = DateTime.UtcNow
        });

        result.Id.ShouldNotBe(Guid.Empty);
        result.Name.ShouldBe("Test Book");
        result.Price.ShouldBe(19.99m);
    }

    [Fact]
    public async Task Should_Not_Create_Book_With_Empty_Name()
    {
        var input = new CreateBookDto { Name = "", Price = 10m };

        await Should.ThrowAsync<AbpValidationException>(async () =>
        {
            await _bookAppService.CreateAsync(input);
        });
    }
}
```

## Checklist for New Features

- Entity created with proper constructors (primary + protected parameterless)
- Constants and enums in Domain.Shared
- Custom repository interface in Domain (only if custom queries needed)
- EF Core DbSet added to DbContext
- Entity configuration with `ConfigureByConvention()`
- Custom repository implementation (only if interface defined)
- Migration generated and applied via DbMigrator
- DTOs created in Application.Contracts (separate input per method)
- Service interface defined with proper naming conventions
- Service implementation with `[Authorize]` (use `virtual` for reusable modules)
- Object mapping configured (Mapperly or AutoMapper)
- Localization keys added to JSON files
- Permissions defined and registered
- Tests written for happy path and error cases
