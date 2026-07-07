---
name: abp-testing
description: ABP testing patterns - integration tests over unit tests, GetRequiredService, IDataSeedContributor, Shouldly assertions, AddAlwaysAllowAuthorization, NSubstitute mocking, WithUnitOfWorkAsync. Use when writing or reviewing tests for application services, domain services, or repositories in ABP projects.
---

# ABP Testing Patterns

ABP recommends **integration tests over unit tests**. Tests run with real services and an in-memory SQLite database. Each test gets a fresh database instance.

## Test Project Structure

| Project | Purpose | Base Class | Tests |
|---------|---------|------------|-------|
| `*.Domain.Tests` | Domain logic, entities, domain services | `*DomainTestBase` | Business rules, entity invariants |
| `*.Application.Tests` | Application services | `*ApplicationTestBase` | Use case orchestration, authorization |
| `*.EntityFrameworkCore.Tests` | Repository implementations | `*EntityFrameworkCoreTestBase` | Data access, queries |
| `*.MongoDB.Tests` | MongoDB repository implementations | `*MongoDbTestBase` | MongoDB data access |

## Integration Test Approach

- Tests run with **real services** and **real database** (SQLite in-memory for EF Core)
- No mocking of internal ABP services
- Each test gets a **fresh database** instance (isolated)
- Use `GetRequiredService<T>()` to resolve services from DI

## Application Service Test

```csharp
public class BookAppService_Tests : MyProjectApplicationTestBase
{
    private readonly IBookAppService _bookAppService;

    public BookAppService_Tests()
    {
        _bookAppService = GetRequiredService<IBookAppService>();
    }

    [Fact]
    public async Task Should_Get_List_Of_Books()
    {
        // Act
        var result = await _bookAppService.GetListAsync(
            new PagedAndSortedResultRequestDto()
        );

        // Assert
        result.TotalCount.ShouldBeGreaterThan(0);
        result.Items.ShouldContain(b => b.Name == "Test Book");
    }

    [Fact]
    public async Task Should_Create_Book()
    {
        // Arrange
        var input = new CreateBookDto
        {
            Name = "New Book",
            Price = 19.99m,
            AuthorId = Guid.NewGuid(),
            PublishDate = DateTime.UtcNow
        };

        // Act
        var result = await _bookAppService.CreateAsync(input);

        // Assert
        result.Id.ShouldNotBe(Guid.Empty);
        result.Name.ShouldBe("New Book");
        result.Price.ShouldBe(19.99m);
    }

    [Fact]
    public async Task Should_Not_Create_Book_With_Invalid_Name()
    {
        // Arrange
        var input = new CreateBookDto { Name = "", Price = 10m };

        // Act & Assert
        await Should.ThrowAsync<AbpValidationException>(async () =>
        {
            await _bookAppService.CreateAsync(input);
        });
    }

    [Fact]
    public async Task Should_Not_Allow_Unauthorized_Access()
    {
        // Test authorization by checking if unauthorized user can access
        // (requires setting up test without AddAlwaysAllowAuthorization)
    }
}
```

## Domain Service Test

```csharp
public class BookManager_Tests : MyProjectDomainTestBase
{
    private readonly BookManager _bookManager;
    private readonly IBookRepository _bookRepository;

    public BookManager_Tests()
    {
        _bookManager = GetRequiredService<BookManager>();
        _bookRepository = GetRequiredService<IBookRepository>();
    }

    [Fact]
    public async Task Should_Create_Book()
    {
        // Act
        var book = await _bookManager.CreateAsync("Test Book", 29.99m);

        // Assert
        book.ShouldNotBeNull();
        book.Name.ShouldBe("Test Book");
        book.Price.ShouldBe(29.99m);
    }

    [Fact]
    public async Task Should_Not_Allow_Duplicate_Book_Name()
    {
        // Arrange
        await _bookManager.CreateAsync("Existing Book", 10m);

        // Act & Assert
        var exception = await Should.ThrowAsync<BusinessException>(async () =>
        {
            await _bookManager.CreateAsync("Existing Book", 20m);
        });

        exception.Code.ShouldBe("MyProject:BookNameAlreadyExists");
    }
}
```

## Test Naming Convention

Use descriptive names following the pattern `Should_ExpectedBehavior_When_Condition`:

```csharp
public async Task Should_Create_Book_When_Input_Is_Valid()
public async Task Should_Throw_BusinessException_When_Name_Already_Exists()
public async Task Should_Return_Empty_List_When_No_Books_Exist()
public async Task Should_Update_Book_Price_When_Valid_Price_Provided()
public async Task Should_Not_Delete_Book_When_User_Is_Not_Owner()
```

## Arrange-Act-Assert (AAA) Pattern

```csharp
[Fact]
public async Task Should_Update_Book_Price()
{
    // Arrange — set up test data
    var bookId = await CreateTestBookAsync();
    var newPrice = 39.99m;

    // Act — execute the operation under test
    var result = await _bookAppService.UpdateAsync(bookId, new UpdateBookDto
    {
        Name = "Updated Name",
        Price = newPrice
    });

    // Assert — verify the result
    result.Price.ShouldBe(newPrice);
}
```

## Assertions with Shouldly

ABP uses the **Shouldly** assertion library:

```csharp
// Equality
result.ShouldNotBeNull();
result.Name.ShouldBe("Expected Name");
result.Id.ShouldNotBe(Guid.Empty);

// Numeric
result.Price.ShouldBeGreaterThan(0);
result.Price.ShouldBeLessThan(1000);
result.Items.Count.ShouldBe(5);

// Collections
result.Items.ShouldContain(x => x.Id == expectedId);
result.Items.ShouldBeEmpty();
result.Items.ShouldNotBeEmpty();
result.Items.ShouldAllBe(x => x.IsActive);

// Exception assertions
await Should.ThrowAsync<BusinessException>(async () =>
{
    await _service.DoSomethingAsync();
});

// Exception with code check
var ex = await Should.ThrowAsync<BusinessException>(async () =>
{
    await _service.DoSomethingAsync();
});
ex.Code.ShouldBe("MyProject:ErrorCode");

// Exception with data check
ex.Data["Name"].ShouldBe("DuplicateName");
```

## Test Data Seeding

Use `IDataSeedContributor` to seed test data:

```csharp
public class MyProjectTestDataSeedContributor :
    IDataSeedContributor,
    ITransientDependency
{
    public static readonly Guid TestBookId = Guid.Parse("3fa85f64-5717-4562-b3fc-2c963f66afa6");
    public static readonly Guid TestAuthorId = Guid.Parse("4fa85f64-5717-4562-b3fc-2c963f66afa6");

    private readonly IBookRepository _bookRepository;
    private readonly IGuidGenerator _guidGenerator;

    public MyProjectTestDataSeedContributor(
        IBookRepository bookRepository,
        IGuidGenerator guidGenerator)
    {
        _bookRepository = bookRepository;
        _guidGenerator = guidGenerator;
    }

    public async Task SeedAsync(DataSeedContext context)
    {
        await _bookRepository.InsertAsync(
            new Book(TestBookId, "Test Book", 19.99m, TestAuthorId, DateTime.UtcNow),
            autoSave: true
        );
    }
}
```

## Disabling Authorization in Tests

For testing business logic without permission concerns:

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddAlwaysAllowAuthorization();
}
```

> Use this in your test base class when you want to test business logic without setting up permissions. For authorization-specific tests, don't add this.

## Mocking External Services

Use **NSubstitute** for mocking external dependencies:

```csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    // Mock email sender
    var emailSender = Substitute.For<IEmailSender>();
    emailSender.SendAsync(
        Arg.Any<string>(),
        Arg.Any<string>(),
        Arg.Any<string>()
    ).Returns(Task.CompletedTask);

    context.Services.AddSingleton(emailSender);

    // Mock with specific behavior
    var paymentService = Substitute.For<IPaymentService>();
    paymentService.ProcessAsync(Arg.Any<PaymentRequest>())
        .Returns(new PaymentResult { Success = true });

    context.Services.AddSingleton(paymentService);
}
```

## Testing with Specific User

```csharp
[Fact]
public async Task Should_Get_Current_User_Books()
{
    await WithUnitOfWorkAsync(async () =>
    {
        using (CurrentUser.Change(TestData.UserId, TestData.UserName))
        {
            var result = await _bookAppService.GetMyBooksAsync();
            result.Items.ShouldAllBe(b => b.CreatorId == TestData.UserId);
        }
    });
}
```

## Testing Multi-Tenancy

```csharp
[Fact]
public async Task Should_Filter_Books_By_Tenant()
{
    using (CurrentTenant.Change(TestData.TenantId))
    {
        var result = await _bookAppService.GetListAsync(
            new PagedAndSortedResultRequestDto()
        );

        // Results should only contain current tenant's books
        result.Items.ShouldAllBe(b => b.TenantId == TestData.TenantId);
    }
}

[Fact]
public async Task Should_Access_Host_Data()
{
    using (CurrentTenant.Change(null))  // Switch to host context
    {
        var result = await _bookAppService.GetListAsync(
            new PagedAndSortedResultRequestDto()
        );

        // Results should contain host-level data
    }
}
```

## Best Practices

- Each test should be **independent** — don't share state between tests
- Use **meaningful test data** — avoid magic strings/numbers
- Test **edge cases and error conditions** — not just happy path
- Keep tests **focused on single behavior** — one assertion concept per test
- Use **test data seeders** for common/reusable test data
- Avoid testing **framework internals** — test your business logic
- Use `WithUnitOfWorkAsync()` when manually controlling the UoW scope
- Prefer **integration tests** over unit tests in ABP

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| Mocking ABP internal services | Tests become brittle, don't test real behavior | Use real services with in-memory DB |
| Sharing state between tests | Tests become order-dependent | Each test creates its own data |
| Testing framework code | Wastes time, tests break on upgrades | Test your business logic only |
| Not testing error cases | Only happy path covered | Test validation, authorization, business rule violations |
| Hardcoded test data without constants | Magic values scattered across tests | Use `TestData` static class with constants |

## Best Practices Checklist

- Tests inherit from appropriate test base class
- `GetRequiredService<T>()` used for service resolution
- Shouldly assertions used consistently
- AAA pattern followed (Arrange-Act-Assert)
- Test naming follows `Should_ExpectedBehavior_When_Condition`
- Test data seeded via `IDataSeedContributor`
- `AddAlwaysAllowAuthorization()` used when not testing auth
- NSubstitute used for external service mocking
- `WithUnitOfWorkAsync()` used for manual UoW control
- Both host and tenant contexts tested (if multi-tenant)
- Error cases and edge cases covered
