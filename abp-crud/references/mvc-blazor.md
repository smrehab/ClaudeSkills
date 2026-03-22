# ABP CRUD: MVC Razor Pages & Blazor UI Patterns

## MVC / Razor Pages

### Generate Pages with ABP CLI
```bash
abp generate-crud-page --entity {Entity} --namespace {RootNamespace}.{Feature}
```
This scaffolds: Index.cshtml, CreateModal.cshtml, EditModal.cshtml.

### Manual Pattern — Index Page

`Pages/{Feature}/Index.cshtml.cs`:
```csharp
public class IndexModel : {App}PageModel
{
    public IndexModel() {}
}
```

`Pages/{Feature}/Index.cshtml`:
```html
@page
@model IndexModel
@{
    ViewData["Title"] = "{Feature}";
}
<abp-card>
    <abp-card-header>
        <abp-row>
            <abp-column size-md="_6">
                <abp-card-title>{Feature}</abp-card-title>
            </abp-column>
            <abp-column size-md="_6" class="text-end">
                <abp-button id="New{Entity}Button"
                    text="New {Entity}"
                    icon="plus"
                    button-type="Primary"
                    permission="{App}.{Feature}.Create" />
            </abp-column>
        </abp-row>
    </abp-card-header>
    <abp-card-body>
        <abp-table striped-rows="true" id="{Entity}Table"></abp-table>
    </abp-card-body>
</abp-card>
```

### JavaScript for MVC DataTable

```javascript
// wwwroot/pages/{feature}/index.js
$(function () {
    var l = abp.localization.getResource('{App}');
    var service = {rootNamespace}.{feature}.{entity};

    var dataTable = $('#{Entity}Table').DataTable(
        abp.libs.datatables.normalizeConfiguration({
            serverSide: true,
            paging: true,
            order: [[1, "asc"]],
            searching: false,
            ajax: abp.libs.datatables.createAjax(service.getList),
            columnDefs: [
                {
                    title: l('{Entity}Name'),
                    data: "name"
                },
                {
                    title: l('Actions'),
                    rowAction: {
                        items: [
                            {
                                text: l('Edit'),
                                visible: abp.auth.isGranted('{App}.{Feature}.Update'),
                                action: function (data) {
                                    editModal.open({ id: data.record.id });
                                }
                            },
                            {
                                text: l('Delete'),
                                visible: abp.auth.isGranted('{App}.{Feature}.Delete'),
                                confirmMessage: function (data) {
                                    return l('Are you sure to delete: {0}?', data.record.name);
                                },
                                action: function (data) {
                                    service.delete(data.record.id)
                                        .then(function () {
                                            abp.notify.info(l('SuccessfullyDeleted'));
                                            dataTable.ajax.reload();
                                        });
                                }
                            }
                        ]
                    }
                }
            ]
        })
    );

    var createModal = new abp.ModalManager(abp.appPath + '{Feature}/CreateModal');
    var editModal = new abp.ModalManager(abp.appPath + '{Feature}/EditModal');

    createModal.onResult(function () { dataTable.ajax.reload(); });
    editModal.onResult(function () { dataTable.ajax.reload(); });

    $('#New{Entity}Button').click(function (e) {
        e.preventDefault();
        createModal.open();
    });
});
```

### Anti-Forgery Token (MVC — critical)

If there is **no** `_ViewImports.cshtml` in the Pages directory, the `asp-antiforgery="true"` tag helper is NOT processed → POST returns 400. Use `@Html.AntiForgeryToken()` explicitly inside the `<form>` tag instead of relying on the tag helper:

```html
<form method="post">
    @Html.AntiForgeryToken()
    <!-- fields -->
</form>
```

---

### Create Modal (MVC)

`Pages/{Feature}/CreateModal.cshtml.cs`:
```csharp
public class CreateModalModel : {App}PageModel
{
    [BindProperty]
    public Create{Entity}Dto {Entity} { get; set; }

    private readonly I{Entity}AppService _service;

    public CreateModalModel(I{Entity}AppService service)
    {
        _service = service;
    }

    public void OnGet() { {Entity} = new Create{Entity}Dto(); }

    public async Task<IActionResult> OnPostAsync()
    {
        await _service.CreateAsync({Entity});
        return NoContent();
    }
}
```

---

## Blazor (Server-Side or WebAssembly)

### Component Pattern

```razor
@page "/{feature}"
@using {RootNamespace}.{Feature}
@inject I{Entity}AppService AppService
@inject IUiMessageService UiMessageService

<h1>{Feature}</h1>

@if (Items == null)
{
    <p>Loading...</p>
}
else
{
    <Button Color="Color.Primary" Clicked="OpenCreateModal">New {Entity}</Button>

    <DataGrid TItem="{Entity}Dto"
              Data="Items"
              ReadData="OnReadData"
              TotalItems="TotalCount"
              ShowPager="true"
              PageSize="10">
        <DataGridColumn TItem="{Entity}Dto" Field="@nameof({Entity}Dto.Name)" Caption="Name" />
        <DataGridColumn TItem="{Entity}Dto">
            <DisplayTemplate>
                <Button Color="Color.Warning" Clicked="() => OpenEditModal(context)">Edit</Button>
                <Button Color="Color.Danger" Clicked="() => Delete(context.Id)">Delete</Button>
            </DisplayTemplate>
        </DataGridColumn>
    </DataGrid>
}

@code {
    IReadOnlyList<{Entity}Dto>? Items;
    int TotalCount;
    int CurrentPage = 1;
    int PageSize = 10;

    protected override async Task OnInitializedAsync()
    {
        await GetEntitiesAsync();
    }

    private async Task GetEntitiesAsync()
    {
        var result = await AppService.GetListAsync(new Get{Entity}ListInput
        {
            MaxResultCount = PageSize,
            SkipCount = (CurrentPage - 1) * PageSize
        });
        Items = result.Items;
        TotalCount = (int)result.TotalCount;
    }

    private async Task OnReadData(DataGridReadDataEventArgs<{Entity}Dto> e)
    {
        CurrentPage = e.Page;
        await GetEntitiesAsync();
    }

    private async Task Delete(Guid id)
    {
        var confirmed = await UiMessageService.Confirm("Are you sure?");
        if (!confirmed) return;
        await AppService.DeleteAsync(id);
        await GetEntitiesAsync();
    }

    // Modal state and handlers (create/edit) omitted for brevity
    private void OpenCreateModal() { /* open modal */ }
    private void OpenEditModal({Entity}Dto entity) { /* open modal with entity */ }
}
```

### Blazor Permission Check

```razor
@inject IAuthorizationService AuthorizationService

@if (await AuthorizationService.IsGrantedAsync("{App}.{Feature}.Create"))
{
    <Button Color="Color.Primary" Clicked="OpenCreateModal">New {Entity}</Button>
}
```

Or use the `<AuthorizeView>` component:
```razor
<AuthorizeView Policy="{App}.{Feature}.Create">
    <Button>New {Entity}</Button>
</AuthorizeView>
```
