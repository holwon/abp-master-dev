---
name: abp-blazor
description: ABP Blazor UI patterns - AbpComponentBase, AbpCrudPageBase, DataGrid, IMenuContributor, Message/Notify, Validations, JavaScript interop. Use when building or reviewing Blazor Server or WebAssembly UI components in ABP projects.
---

# ABP Blazor UI

ABP provides a complete Blazor UI infrastructure with base classes, pre-built components, and services for both Blazor Server and WebAssembly.

## Component Base Classes

### AbpComponentBase — Basic Component

All Blazor components should inherit from `AbpComponentBase` to access localization, authorization, and other ABP services:

```razor
@inherits AbpComponentBase

<h1>@L["Books"]</h1>
```

### AbpCrudPageBase — CRUD Page

For full CRUD pages, inherit from `AbpCrudPageBase` which provides:
- Entity list management (`Entities`, `TotalCount`, `PageSize`)
- Create/Edit modal lifecycle (`OpenCreateModalAsync`, `OpenEditModalAsync`)
- Delete with confirmation (`DeleteEntityAsync`)
- Permission checks (`HasCreatePermission`, `HasUpdatePermission`, `HasDeletePermission`)

```razor
@page "/books"
@inherits AbpCrudPageBase<IBookAppService, BookDto, Guid, PagedAndSortedResultRequestDto, CreateUpdateBookDto>

<Card>
    <CardHeader>
        <Row>
            <Column>
                <h2>@L["Books"]</h2>
            </Column>
            <Column TextAlignment="TextAlignment.End">
                @if (HasCreatePermission)
                {
                    <Button Color="Color.Primary" Clicked="OpenCreateModalAsync">
                        @L["NewBook"]
                    </Button>
                }
            </Column>
        </Row>
    </CardHeader>
    <CardBody>
        <DataGrid TItem="BookDto"
                  Data="Entities"
                  ReadData="OnDataGridReadAsync"
                  TotalItems="TotalCount"
                  ShowPager="true"
                  PageSize="PageSize">
            <DataGridColumns>
                <DataGridColumn Field="@nameof(BookDto.Name)" Caption="@L["Name"]" />
                <DataGridColumn Field="@nameof(BookDto.Price)" Caption="@L["Price"]" />
                <DataGridEntityActionsColumn TItem="BookDto">
                    <DisplayTemplate>
                        <EntityActions TItem="BookDto">
                            <EntityAction TItem="BookDto"
                                          Text="@L["Edit"]"
                                          Visible="HasUpdatePermission"
                                          Clicked="() => OpenEditModalAsync(context)" />
                            <EntityAction TItem="BookDto"
                                          Text="@L["Delete"]"
                                          Visible="HasDeletePermission"
                                          Clicked="() => DeleteEntityAsync(context)"
                                          ConfirmationMessage="() => GetDeleteConfirmationMessage(context)" />
                        </EntityActions>
                    </DisplayTemplate>
                </DataGridEntityActionsColumn>
            </DataGridColumns>
        </DataGrid>
    </CardBody>
</Card>
```

### DataGrid Component

`DataGrid` provides server-side pagination, sorting, and filtering:

| Property | Description |
|----------|-------------|
| `TItem` | Entity DTO type |
| `Data` | Current page items |
| `ReadData` | Callback for loading data |
| `TotalItems` | Total record count |
| `ShowPager` | Show pagination controls |
| `PageSize` | Records per page |

## Localization

```razor
@* Using L property from AbpComponentBase *@
<h1>@L["PageTitle"]</h1>

@* With parameters *@
<p>@L["WelcomeMessage", CurrentUser.UserName]</p>

@* In code-behind *@
var localizedText = L["Save"];
```

## Authorization

### Declarative in Razor

```razor
@* Check permission before rendering *@
@if (await AuthorizationService.IsGrantedAsync("MyPermission"))
{
    <Button Color="Color.Primary">Admin Action</Button>
}

@* Using AuthorizeView for policy-based auth *@
<AuthorizeView Policy="MyPolicy">
    <Authorized>
        <p>You have access!</p>
    </Authorized>
    <NotAuthorized>
        <p>Access denied.</p>
    </NotAuthorized>
</AuthorizeView>
```

### Programmatic in Code-Behind

```csharp
if (await AuthorizationService.IsGrantedAsync("BookStore.Books.Create"))
{
    // Show create UI
}
```

## Navigation & Menu (IMenuContributor)

Configure navigation in `*MenuContributor.cs`:

```csharp
public class MyMenuContributor : IMenuContributor
{
    public async Task ConfigureMenuAsync(MenuConfigurationContext context)
    {
        if (context.Menu.Name == StandardMenus.Main)
        {
            var bookMenu = new ApplicationMenuItem(
                "Books",
                l["Menu:Books"],
                "/books",
                icon: "fa fa-book"
            );

            if (await context.IsGrantedAsync(MyPermissions.Books.Default))
            {
                context.Menu.AddItem(bookMenu);
            }
        }
    }
}
```

## Notifications & Messages

```csharp
// Success message (toast)
await Message.Success(L["BookCreatedSuccessfully"]);

// Error message
await Message.Error(L["OperationFailed"]);

// Confirmation dialog
if (await Message.Confirm(L["AreYouSure"], L["DeleteConfirmation"]))
{
    // User confirmed — proceed with delete
}

// Toast notification (non-blocking)
await Notify.Success(L["OperationCompleted"]);
await Notify.Error(L["SomethingWentWrong"]);
await Notify.Info(L["Processing"]);
await Notify.Warn(L["PleaseReview"]);
```

## Forms & Validation

```razor
<Form @ref="CreateForm">
    <Validations @ref="CreateValidationsRef" Model="@NewEntity" ValidateOnLoad="false">
        <Validation MessageLocalizer="@LH.Localize">
            <Field>
                <FieldLabel>@L["Name"]</FieldLabel>
                <TextEdit @bind-Text="@NewEntity.Name">
                    <Feedback>
                        <ValidationError />
                    </Feedback>
                </TextEdit>
            </Field>
            <Field>
                <FieldLabel>@L["Price"]</FieldLabel>
                <NumericEdit @bind-Value="@NewEntity.Price">
                    <Feedback>
                        <ValidationError />
                    </Feedback>
                </NumericEdit>
            </Field>
        </Validation>
    </Validations>
</Form>

<Button Color="Color.Primary" Clicked="SaveAsync">@L["Save"]</Button>
```

```csharp
@code {
    private Validations CreateValidationsRef;
    private CreateUpdateBookDto NewEntity = new();

    private async Task SaveAsync()
    {
        if (await CreateValidationsRef.ValidateAll())
        {
            await BookAppService.CreateAsync(NewEntity);
            await Message.Success(L["BookCreated"]);
        }
    }
}
```

## JavaScript Interop

```csharp
@inject IJSRuntime JsRuntime

@code {
    private async Task CallJavaScript()
    {
        await JsRuntime.InvokeVoidAsync("myFunction", arg1, arg2);
        var result = await JsRuntime.InvokeAsync<string>("myFunctionWithReturn");
    }
}
```

## State Management

Inject service proxies from `HttpApi.Client`:

```csharp
@inject IBookAppService BookAppService

@code {
    private List<BookDto> Books { get; set; } = new();

    protected override async Task OnInitializedAsync()
    {
        var result = await BookAppService.GetListAsync(
            new PagedAndSortedResultRequestDto()
        );
        Books = result.Items.ToList();
    }
}
```

## Code-Behind Pattern

Separate UI markup from logic using partial classes:

**Books.razor:**
```razor
@page "/books"
@inherits BooksBase

<h1>@L["Books"]</h1>
```

**Books.razor.cs:**
```csharp
public partial class Books : BooksBase
{
    // Component-specific logic here
}
```

**BooksBase.cs:**
```csharp
public abstract class BooksBase : AbpComponentBase
{
    [Inject]
    protected IBookAppService BookAppService { get; set; }

    protected List<BookDto> Books { get; set; } = new();
}
```

## Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|-----------------|
| Not inheriting from `AbpComponentBase` | No access to L[], auth, etc. | Always inherit from ABP base class |
| Direct HTTP calls instead of proxies | No type safety, no auth handling | Use `I*AppService` from HttpApi.Client |
| Hardcoded strings in Razor | Not localizable | Use `L["Key"]` |
| Checking permissions only in UI | Client-side check can be bypassed | Always check on server too |
| Not using `AbpCrudPageBase` for CRUD | Reinventing pagination, modals, etc. | Use `AbpCrudPageBase` for CRUD pages |

## Best Practices Checklist

- Components inherit from `AbpComponentBase` or `AbpCrudPageBase`
- `L["Key"]` used for all user-facing strings
- `AuthorizationService.IsGrantedAsync()` for permission checks
- `IMenuContributor` for navigation configuration
- `Message.Success()` / `Message.Confirm()` for user interaction
- `Notify.Success()` for non-blocking notifications
- `Validations` component for form validation
- Service proxies from `HttpApi.Client` for data access
- Code-behind pattern for separation of concerns
- `DataGrid` with `ReadData` for server-side pagination
