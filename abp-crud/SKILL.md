---
name: abp-crud
description: >
  Full CRUD feature implementation for ABP Framework Community Edition.
  Use this skill whenever the user wants to create a new entity, add CRUD for a feature,
  implement a domain object, or build any new data model with ABP. Triggers on phrases like
  "create entity X", "add X CRUD", "implement X module", "need X feature", "add X to the app",
  "create X with fields/functions", or any request to build a new backend feature from scratch.
  Covers all layers in order: Domain → Application Contracts → Application → EF Core → HTTP API → Angular UI.
  Do NOT skip this skill even if the request seems simple — ABP has too many required steps to do from memory.
---

# ABP Full CRUD Implementation

Execute every step in order. Do NOT skip steps. Do NOT consult abp.io docs — this skill encodes the correct patterns. Treat every `{placeholder}` as a fill-in based on the entity being created.

---

## Step 0 — Gather Spec

Present these questions to the user as a numbered list and wait for answers before writing any code. Group them clearly.

### Entity Identity
1. **Entity name** — singular PascalCase (e.g. `Department`, `Resource`, `Invoice`)
2. **Feature folder** — usually the plural (e.g. `Departments`)
3. **App name prefix** — found in existing class names (e.g. `CyberMAKPM`)
4. **Fields** — for each: name, C# type, required/optional, max length, default value

### Domain Design
5. **Multi-tenancy** — is this entity tenant-scoped? (yes for most app entities; no for shared config/lookup tables)
6. **Audit level** — choose one:
   - `None` — no audit fields (rare, e.g. junction tables)
   - `Creation` — tracks who/when created only
   - `Full` — creation + modification + soft-delete ← **default, use unless told otherwise**
7. **Status/type enum?** — yes/no; if yes, list the values
8. **Relationships** — for each related entity: which entity, cardinality (one-to-many / many-to-one / many-to-many), FK direction, cascade behavior

### AppService Operations
9. **Standard CRUD** — GetList, Get, Create, Update, Delete (always included)
10. **Extra operations** — select any that apply:
    - Bulk delete
    - Bulk status update
    - Export (CSV / Excel)
    - Import
    - Lookup list (lightweight, for populating dropdowns — returns `{id, name}` pairs)
    - Sub-collection CRUD (e.g. Comments, Attachments on this entity)
    - Custom domain actions (e.g. Approve, Archive, Assign)

### UI Layer
11. **UI layer** — Angular | MVC | Blazor | API-only
12. **Grid/UI library** — Syncfusion Grid | Syncfusion TreeGrid (hierarchical) | Angular Material | plain Bootstrap | other
13. **List features** — which of these: search/filter bar | server-side paging | column sort | column resize | export button | row selection
14. **Create/Edit UI** — Modal dialog form | Separate page/route form | Inline grid editing
15. **Routing** — what route path (e.g. `/departments`), which parent route/module

### Implementation Options
16. **Object mapper** — Mapperly (new default) | AutoMapper (use if the existing project already uses it — don't mix both)
17. **Localization** — add localization keys for labels and messages? (yes / no — skip for internal/admin tools)
18. **Tests** — generate integration test class? (yes / no)

You have everything you need once you can fill in every `{placeholder}` below. Mark any spec decisions the user left open with a sensible default and note it.

---

## Step 1 — Domain.Shared: Enum (if needed)

*Skip if no status or type enum is required.*

**New file:** `src/{App}.Domain.Shared/{Feature}/{Entity}Status.cs`

```csharp
namespace {RootNamespace}.{Feature};

public enum {Entity}Status
{
    Active = 0,
    Inactive = 1,
    // add lifecycle values per spec
}
```

---

## Step 2 — Domain: Entity

**New file:** `src/{App}.Domain/{Feature}/{Entity}.cs`

Choose the base class from spec question 5–6:

| Spec answer | Base class |
|-------------|-----------|
| No tenancy, No audit | `AggregateRoot<Guid>` |
| No tenancy, Full audit | `FullAuditedAggregateRoot<Guid>` |
| Tenanted, Creation audit only | `CreationAuditedAggregateRoot<Guid>` + `IMultiTenant` |
| Tenanted, Full audit *(default)* | `FullAuditedAggregateRoot<Guid>` + `IMultiTenant` |

```csharp
using System;
using Volo.Abp;
using Volo.Abp.Domain.Entities.Auditing;
using Volo.Abp.MultiTenancy;

namespace {RootNamespace}.{Feature};

// Default: tenanted + full audit. Adjust base class per table above.
public class {Entity} : FullAuditedAggregateRoot<Guid>, IMultiTenant
{
    public Guid? TenantId { get; set; }   // remove if not multi-tenant

    public string Name { get; set; } = null!;      // required string
    public string? Description { get; set; }        // optional string
    public {Entity}Status Status { get; set; }      // if enum applies

    // Foreign key pattern — one per relationship
    public Guid? {Related}Id { get; set; }

    // Navigation property (only if EF Include queries are needed)
    public {Related}? {Related} { get; set; }

    // EF Core requires a protected parameterless constructor
    protected {Entity}() { }

    public {Entity}(Guid id, string name) : base(id)
    {
        Name = Check.NotNullOrWhiteSpace(name, nameof(name), maxLength: 256);
    }
}
```

**DDD Entity Rules (from ABP framework guidelines):**
- Protected parameterless constructor is always required for EF Core.
- Use `Check.NotNullOrWhiteSpace()` for required string constructor args.
- Keep navigation properties as nullable; EF will populate them when `.Include()` is used.
- **Rich model, not anemic**: use private/protected setters + domain methods for properties with business rules. Use public setters only for simple data properties that have no invariants.
- **Never generate GUIDs inside the entity** — receive `id` as a constructor parameter, generated externally by `GuidGenerator.Create()`.
- **Never reference other aggregate roots by navigation property** — use `Guid? RelatedId` only. Navigation to child entities within the same aggregate is fine.
- Use `Clock.Now` for any date-time default values (never `DateTime.Now` — Clock is timezone-aware and testable).

```csharp
// Example: rich entity with domain method
public class {Entity} : FullAuditedAggregateRoot<Guid>, IMultiTenant
{
    public Guid? TenantId { get; set; }
    public string Name { get; private set; } = null!;   // private setter — changed via domain method
    public {Entity}Status Status { get; private set; }
    public DateTime? StartDate { get; set; }             // public setter — no invariant

    protected {Entity}() { }

    public {Entity}(Guid id, string name) : base(id)
    {
        SetName(name);
        Status = {Entity}Status.Active;
    }

    public void SetName(string name)
    {
        Name = Check.NotNullOrWhiteSpace(name, nameof(name), maxLength: 256);
    }

    public void Activate()   { Status = {Entity}Status.Active; }
    public void Deactivate() { Status = {Entity}Status.Inactive; }
}
```

---

## Step 3 — Domain: Repository Interface

*Skip if basic Get/GetList/Insert/Update/Delete is sufficient — those come free from `IRepository<T, TKey>`. Add a custom interface only when filtering, searching, or joining is needed.*

**New file:** `src/{App}.Domain/{Feature}/I{Entity}Repository.cs`

```csharp
namespace {RootNamespace}.{Feature};

public interface I{Entity}Repository : IRepository<{Entity}, Guid>
{
    Task<List<{Entity}>> GetFilteredListAsync(
        string? filter = null,
        Guid? {relatedId} = null,          // one param per FK filter from spec
        {Entity}Status? status = null,     // if status enum exists
        string? sorting = null,
        int maxResultCount = int.MaxValue,
        int skipCount = 0,
        CancellationToken cancellationToken = default);

    Task<long> GetFilteredCountAsync(
        string? filter = null,
        Guid? {relatedId} = null,
        {Entity}Status? status = null,
        CancellationToken cancellationToken = default);

    // If Lookup operation requested:
    Task<List<{Entity}>> GetLookupListAsync(CancellationToken cancellationToken = default);
}
```

---

## Step 4 — Application.Contracts: DTOs

**New file:** `src/{App}.Application.Contracts/{Feature}/{Entity}Dto.cs`

```csharp
namespace {RootNamespace}.{Feature};

// Output DTO — mirrors entity + denormalized display fields
public class {Entity}Dto : FullAuditedEntityDto<Guid>  // or AuditedEntityDto / EntityDto per audit level
{
    public string Name { get; set; } = null!;
    public {Entity}Status Status { get; set; }         // if enum exists
    public Guid? {Related}Id { get; set; }
    public string? {Related}Name { get; set; }          // denormalized for display — populated in AppService
    // mirror all other entity properties
}

// Create input
public class Create{Entity}Dto
{
    [Required]
    [StringLength(256)]
    public string Name { get; set; } = null!;
    public {Entity}Status Status { get; set; } = {Entity}Status.Active;
    public Guid? {Related}Id { get; set; }
    // all settable fields; NO ConcurrencyStamp
}

// Update input
public class Update{Entity}Dto
{
    [Required]
    [StringLength(256)]
    public string Name { get; set; } = null!;
    public {Entity}Status Status { get; set; }
    public Guid? {Related}Id { get; set; }

    [Required]
    public string ConcurrencyStamp { get; set; } = null!;  // always required for optimistic locking
}

// List query input
public class Get{Entity}ListInput : PagedAndSortedResultRequestDto
{
    public string? Filter { get; set; }
    public Guid? {Related}Id { get; set; }
    public {Entity}Status? Status { get; set; }
}

// Lookup DTO (only if Lookup operation requested)
public class {Entity}LookupDto
{
    public Guid Id { get; set; }
    public string Name { get; set; } = null!;
}
```

---

## Step 5 — Application.Contracts: AppService Interface

**New file:** `src/{App}.Application.Contracts/{Feature}/I{Entity}AppService.cs`

```csharp
namespace {RootNamespace}.{Feature};

public interface I{Entity}AppService : IApplicationService
{
    // Standard CRUD — always present
    Task<PagedResultDto<{Entity}Dto>> GetListAsync(Get{Entity}ListInput input);
    Task<{Entity}Dto> GetAsync(Guid id);
    Task<{Entity}Dto> CreateAsync(Create{Entity}Dto input);
    Task<{Entity}Dto> UpdateAsync(Guid id, Update{Entity}Dto input);
    Task DeleteAsync(Guid id);

    // Add only what was requested in spec:
    // Task<List<{Entity}LookupDto>> GetLookupListAsync();
    // Task<BulkOperationResultDto> BulkDeleteAsync(List<Guid> ids);
    // Task<IRemoteStreamContent> ExportAsync();
}
```

---

## Step 6 — Application.Contracts: Permissions

**Edit** `src/{App}.Application.Contracts/Permissions/{App}Permissions.cs` — add nested class:

```csharp
public static class {Feature}
{
    public const string Default = GroupName + ".{Feature}";
    public const string View   = Default + ".View";
    public const string Create = Default + ".Create";
    public const string Update = Default + ".Update";
    public const string Delete = Default + ".Delete";
    // public const string Export = Default + ".Export";    // if export requested
    // public const string BulkDelete = Default + ".BulkDelete";  // if bulk delete requested
}
```

**Edit** `src/{App}.Application.Contracts/Permissions/{App}PermissionDefinitionProvider.cs`:

```csharp
var {feature} = group.AddPermission(
    {App}Permissions.{Feature}.Default, new FixedLocalizableString("{Feature}"));
{feature}.AddChild({App}Permissions.{Feature}.View,   new FixedLocalizableString("View {Feature}"));
{feature}.AddChild({App}Permissions.{Feature}.Create, new FixedLocalizableString("Create {Feature}"));
{feature}.AddChild({App}Permissions.{Feature}.Update, new FixedLocalizableString("Update {Feature}"));
{feature}.AddChild({App}Permissions.{Feature}.Delete, new FixedLocalizableString("Delete {Feature}"));
// add child permissions for any extras (Export, BulkDelete, etc.)
```

---

## Step 7 — Application: AppService Implementation

**New file:** `src/{App}.Application/{Feature}/{Entity}AppService.cs`

```csharp
[Authorize({App}Permissions.{Feature}.Default)]
public class {Entity}AppService : ApplicationService, I{Entity}AppService
{
    private readonly I{Entity}Repository _repository;
    // inject additional repositories for denormalized fields:
    // private readonly IRepository<{Related}, Guid> _{related}Repository;

    public {Entity}AppService(I{Entity}Repository repository)
    {
        _repository = repository;
    }

    [Authorize({App}Permissions.{Feature}.View)]
    public async Task<PagedResultDto<{Entity}Dto>> GetListAsync(Get{Entity}ListInput input)
    {
        var totalCount = await _repository.GetFilteredCountAsync(
            input.Filter, input.{Related}Id, input.Status);
        var items = await _repository.GetFilteredListAsync(
            input.Filter, input.{Related}Id, input.Status,
            input.Sorting, input.MaxResultCount, input.SkipCount);
        var dtos = ObjectMapper.Map<List<{Entity}>, List<{Entity}Dto>>(items);
        await EnrichDtosAsync(dtos);   // populate denormalized fields
        return new PagedResultDto<{Entity}Dto>(totalCount, dtos);
    }

    public async Task<{Entity}Dto> GetAsync(Guid id)
    {
        var entity = await _repository.GetAsync(id);
        var dto = ObjectMapper.Map<{Entity}, {Entity}Dto>(entity);
        await EnrichDtosAsync(new List<{Entity}Dto> { dto });
        return dto;
    }

    [Authorize({App}Permissions.{Feature}.Create)]
    public async Task<{Entity}Dto> CreateAsync(Create{Entity}Dto input)
    {
        var entity = new {Entity}(GuidGenerator.Create(), input.Name);
        entity.Status = input.Status;
        entity.{Related}Id = input.{Related}Id;
        // set all other properties from input
        await _repository.InsertAsync(entity, autoSave: true);
        return ObjectMapper.Map<{Entity}, {Entity}Dto>(entity);
    }

    [Authorize({App}Permissions.{Feature}.Update)]
    public async Task<{Entity}Dto> UpdateAsync(Guid id, Update{Entity}Dto input)
    {
        var entity = await _repository.GetAsync(id);
        if (entity.ConcurrencyStamp != input.ConcurrencyStamp)
            throw new UserFriendlyException("Record was modified by another user. Please refresh and retry.");
        entity.Name = input.Name;
        entity.Status = input.Status;
        entity.{Related}Id = input.{Related}Id;
        // set all other properties from input
        await _repository.UpdateAsync(entity);
        return ObjectMapper.Map<{Entity}, {Entity}Dto>(entity);
    }

    [Authorize({App}Permissions.{Feature}.Delete)]
    public async Task DeleteAsync(Guid id)
    {
        await _repository.DeleteAsync(id);
    }

    // Lookup (if requested)
    public async Task<List<{Entity}LookupDto>> GetLookupListAsync()
    {
        var items = await _repository.GetLookupListAsync();
        return items.Select(x => new {Entity}LookupDto { Id = x.Id, Name = x.Name }).ToList();
    }

    // Batch-fetch related entity names to avoid N+1 queries
    private async Task EnrichDtosAsync(List<{Entity}Dto> dtos)
    {
        var relatedIds = dtos
            .Where(d => d.{Related}Id.HasValue)
            .Select(d => d.{Related}Id!.Value)
            .Distinct().ToList();
        if (!relatedIds.Any()) return;

        var related = await _{related}Repository.GetListAsync(x => relatedIds.Contains(x.Id));
        var lookup = related.ToDictionary(x => x.Id);
        foreach (var dto in dtos)
            if (dto.{Related}Id.HasValue && lookup.TryGetValue(dto.{Related}Id.Value, out var r))
                dto.{Related}Name = r.Name;
    }
}
```

**Edit** `src/{App}.Application/{App}ApplicationAutoMapperProfile.cs` (AutoMapper only):

```csharp
CreateMap<{Entity}, {Entity}Dto>();
```

**AppService base class properties — available without injection:**

| Property | Type | Use for |
|----------|------|---------|
| `GuidGenerator` | `IGuidGenerator` | `GuidGenerator.Create()` for new entity IDs |
| `Clock` | `IClock` | `Clock.Now` instead of `DateTime.Now` |
| `CurrentUser` | `ICurrentUser` | `CurrentUser.Id`, `CurrentUser.UserName` |
| `CurrentTenant` | `ICurrentTenant` | `CurrentTenant.Id` for tenant-aware logic |
| `ObjectMapper` | `IObjectMapper` | `ObjectMapper.Map<>()` |
| `L` | `IStringLocalizer` | Localized messages |
| `UnitOfWorkManager` | — | Manual transaction control |

Do NOT inject these via constructor — they're inherited from `ApplicationService`.

**AppService rules (from ABP guidelines):**
- Don't repeat the entity name in method names: `GetAsync` not `Get{Entity}Async`.
- Don't put business logic here — call domain methods on the entity or use a domain service.
- Don't call sibling app services within the same module — call repositories directly.
- Throw `BusinessException` for domain-rule violations, `UserFriendlyException` for client-facing messages.
- Use `Clock.Now` not `DateTime.Now` anywhere date/time is needed.

---

## Step 7.5 — Application: Mapperly Mapper (if Mapperly chosen)

*Skip if the project uses AutoMapper. Use Mapperly for new projects — it's the ABP default.*

**New file:** `src/{App}.Application/{Feature}/{Entity}ObjectMapper.cs`

```csharp
using Riok.Mapperly.Abstractions;

namespace {RootNamespace}.{Feature};

[Mapper]
public partial class {Entity}ObjectMapper
{
    public partial {Entity}Dto ToDto({Entity} entity);
    public partial List<{Entity}Dto> ToDtoList(List<{Entity}> entities);
}
```

Register in the Application module:

```csharp
context.Services.AddSingleton<{Entity}ObjectMapper>();
```

Inject into the AppService:

```csharp
private readonly {Entity}ObjectMapper _mapper;

// In GetListAsync:
var dtos = _mapper.ToDtoList(items);
```

---

## Step 8 — EntityFrameworkCore: Repository Implementation

*Skip if Step 3 was skipped.*

**New file:** `src/{App}.EntityFrameworkCore/Repositories/{Entity}Repository.cs`

```csharp
public class {Entity}Repository
    : EfCoreRepository<{App}DbContext, {Entity}, Guid>, I{Entity}Repository
{
    public {Entity}Repository(IDbContextProvider<{App}DbContext> dbContextProvider)
        : base(dbContextProvider) { }

    public async Task<List<{Entity}>> GetFilteredListAsync(
        string? filter = null,
        Guid? {relatedId} = null,
        {Entity}Status? status = null,
        string? sorting = null,
        int maxResultCount = int.MaxValue,
        int skipCount = 0,
        CancellationToken cancellationToken = default)
    {
        // Use GetCancellationToken() — ensures the token is linked to the ABP request cancellation token
        var ct = GetCancellationToken(cancellationToken);
        return await (await GetDbSetAsync())
            .WhereIf(!string.IsNullOrWhiteSpace(filter), x => x.Name.Contains(filter!))
            .WhereIf({relatedId}.HasValue, x => x.{Related}Id == {relatedId}!.Value)
            .WhereIf(status.HasValue, x => x.Status == status!.Value)
            .OrderBy(sorting.IsNullOrWhiteSpace() ? nameof({Entity}.CreationTime) + " desc" : sorting)
            .PageBy(skipCount, maxResultCount)
            .ToListAsync(ct);
    }

    public async Task<long> GetFilteredCountAsync(
        string? filter = null,
        Guid? {relatedId} = null,
        {Entity}Status? status = null,
        CancellationToken cancellationToken = default)
    {
        var ct = GetCancellationToken(cancellationToken);
        return await (await GetDbSetAsync())
            .WhereIf(!string.IsNullOrWhiteSpace(filter), x => x.Name.Contains(filter!))
            .WhereIf({relatedId}.HasValue, x => x.{Related}Id == {relatedId}!.Value)
            .WhereIf(status.HasValue, x => x.Status == status!.Value)
            .LongCountAsync(ct);
    }

    // Lookup — lightweight, no filtering, for dropdowns
    public async Task<List<{Entity}>> GetLookupListAsync(CancellationToken cancellationToken = default)
    {
        var ct = GetCancellationToken(cancellationToken);
        return await (await GetDbSetAsync())
            .OrderBy(x => x.Name)
            .ToListAsync(ct);
    }

    // Override to define default eager loading (e.g. always include related entities)
    // public override async Task<IQueryable<{Entity}>> WithDetailsAsync()
    // {
    //     return (await GetQueryableAsync()).Include(x => x.RelatedEntity);
    // }
}
```

**Edit** `src/{App}.EntityFrameworkCore/{App}EntityFrameworkCoreModule.cs`:

```csharp
options.AddRepository<{Entity}, {Entity}Repository>();
```

---

## Step 9 — EntityFrameworkCore: DbContext + Entity Mapping

**Edit** `src/{App}.EntityFrameworkCore/{App}DbContext.cs`:

```csharp
public DbSet<{Entity}> {Entities} { get; set; } = null!;
```

**Edit** `src/{App}.EntityFrameworkCore/{App}DbContextModelCreatingExtensions.cs`:

```csharp
builder.Entity<{Entity}>(b =>
{
    b.ToTable({App}Consts.DbTablePrefix + "{Entities}", {App}Consts.DbSchema);
    b.ConfigureByConvention();  // NEVER omit — sets up audit fields, soft delete, concurrency stamp

    b.Property(x => x.Name).IsRequired().HasMaxLength(256);
    // b.Property(x => x.Description).HasMaxLength(2000);
    // b.Property(x => x.Amount).HasPrecision(18, 2);

    // b.Ignore(x => x.ComputedProperty);  // computed fields not stored in DB

    // Relationships — one block per FK:
    b.HasOne<{Related}>()
     .WithMany()
     .HasForeignKey(x => x.{Related}Id)
     .OnDelete(DeleteBehavior.Restrict);  // use Restrict unless cascading is explicitly needed

    // Indexes — always include TenantId in composite indexes
    b.HasIndex(x => new { x.TenantId, x.Name });
    // b.HasIndex(x => new { x.TenantId, x.{Related}Id });  // index per FK used in filters
});
```

**Relationship rules:**
- One-to-many owned: use `b.OwnsMany()` (the child lives and dies with the parent)
- One-to-many reference: use `b.HasOne().WithMany()` with `DeleteBehavior.Restrict`
- Many-to-many: create a junction entity (`{A}{B}`) and map both sides explicitly

---

## Step 10 — EF Core Migration

ABP templates include `IDesignTimeDbContextFactory` — no `-s` startup-project flag needed:

```bash
cd src/{App}.EntityFrameworkCore
dotnet ef migrations add Added_{Entity}
```

If the above fails with "unable to find startup project", fall back to:

```bash
dotnet ef migrations add Added_{Entity} \
  --project src/{App}.EntityFrameworkCore \
  --startup-project src/{App}.HttpApi.Host
```

Review the generated migration file. Check:
- All expected columns are present
- FK constraints reference the right table
- No unexpected drops or renames to existing tables
- Precision/scale correct for decimal columns

---

## Step 11 — Seed Permissions + Run DbMigrator

**Edit** seed contributor (`src/{App}.DbMigrator/...SampleDataSeedContributor.cs`):

```csharp
var {feature}Perms = new[]
{
    G + ".{Feature}", G + ".{Feature}.View",
    G + ".{Feature}.Create", G + ".{Feature}.Update", G + ".{Feature}.Delete",
    // add extra permission strings if Export/BulkDelete were requested
};
foreach (var perm in {feature}Perms)
    await _permissionManager.SetAsync(perm, RolePermissionValueProvider.ProviderName, "Admin", true);
```

**Always run inside the tenant context block** — check existing `SeedPermissionsAsync()` for the `using (_currentTenant.Change(...))` wrapper. If it exists, add the new lines inside it.

Run DbMigrator (**full build, never `--no-build`**):

```bash
dotnet run --project src/{App}.DbMigrator
```

**Restart the API host** — ABP permission cache is in-memory. New grants are invisible until restart.

---

## Step 12 — HTTP API Controller

Check if `ConventionalControllers.Create(assembly)` is configured → if yes, skip this step.

**New file:** `src/{App}.HttpApi/Controllers/{Entities}Controller.cs`

```csharp
[ApiController]
[Authorize]
[Route("api/app/{entities}")]
public class {Entities}Controller : {App}ControllerBase
{
    private readonly I{Entity}AppService _appService;
    public {Entities}Controller(I{Entity}AppService appService) { _appService = appService; }

    [HttpGet] public Task<PagedResultDto<{Entity}Dto>> GetListAsync([FromQuery] Get{Entity}ListInput input) => _appService.GetListAsync(input);
    [HttpGet("{id}")] public Task<{Entity}Dto> GetAsync(Guid id) => _appService.GetAsync(id);
    [HttpPost] public Task<{Entity}Dto> CreateAsync(Create{Entity}Dto input) => _appService.CreateAsync(input);
    [HttpPut("{id}")] public Task<{Entity}Dto> UpdateAsync(Guid id, Update{Entity}Dto input) => _appService.UpdateAsync(id, input);
    [HttpDelete("{id}")] public Task DeleteAsync(Guid id) => _appService.DeleteAsync(id);

    // [HttpGet("lookup")] public Task<List<{Entity}LookupDto>> GetLookupListAsync() => _appService.GetLookupListAsync();
    // [HttpPost("bulk-delete")] public Task<BulkOperationResultDto> BulkDeleteAsync([FromBody] List<Guid> ids) => _appService.BulkDeleteAsync(ids);
    // [HttpGet("export")] public Task<IActionResult> ExportAsync() => ...
}
```

---

## Step 13 — Build & Verify

```bash
dotnet build
```

Fix all errors before continuing. Common problems:
- Missing `using` statements
- AutoMapper profile not registered (`Configure<AbpAutoMapperOptions>` in Application module)
- Custom repository not registered (`options.AddRepository<>` in EFCore module)
- Permission string typos (strings, no compile-time check)

---

## Step 14 — Angular: Generate Proxy

*Skip if UI ≠ Angular. API host must be running.*

```bash
cd angular
abp generate-proxy -t ng
```

Re-run any time backend DTOs or service signatures change.

---

## Step 15 — Angular: Components + Routing

Read `references/angular-patterns.md` for complete, copy-paste component code based on the spec decisions:
- Which grid library (Syncfusion vs other)
- Which list features (search, paging, sort, export)
- Which form UI (modal dialog vs separate page)

**Routing** — register in the appropriate `routes.ts`:

```typescript
// In parent routes.ts (e.g. app.routes.ts or a feature routes file)
{
  path: '{feature}',        // e.g. 'departments'
  loadComponent: () =>
    import('./{feature}/{feature}-list.component')
      .then(m => m.{Feature}ListComponent),
  canActivate: [authGuard],
  data: { requiredPolicy: '{App}.{Feature}.View' }
}
```

For modal-based create/edit, no extra route is needed — the modal opens from the list component.
For separate page form, add a child route:

```typescript
{
  path: '{feature}',
  children: [
    { path: '', loadComponent: () => import('./{feature}/{feature}-list.component').then(m => m.{Feature}ListComponent) },
    { path: 'new', loadComponent: () => import('./{feature}/{feature}-form.component').then(m => m.{Feature}FormComponent) },
    { path: ':id/edit', loadComponent: () => import('./{feature}/{feature}-form.component').then(m => m.{Feature}FormComponent) },
  ]
}
```

---

## Critical Gotchas

| # | Rule | Why |
|---|------|-----|
| 1 | Permissions need definition **AND** seeding | Definition registers; seeding grants to Admin. Missing either → 403. |
| 2 | Restart API host after DbMigrator | Permission cache is in-memory. New grants invisible until restart. |
| 3 | `b.ConfigureByConvention()` is mandatory | Wires audit columns, soft-delete, concurrency stamp. Omitting = broken schema. |
| 4 | Always check `ConcurrencyStamp` in `UpdateAsync` | Prevents silent overwrites on concurrent edits. |
| 5 | Protected parameterless constructor in every entity | EF Core materializes via reflection. Without it: runtime crash. |
| 6 | `autoSave: true` on `InsertAsync` | Ensures record is committed immediately in the same unit of work. |
| 7 | DbMigrator: never `--no-build` | Stale build → wrong migration → schema drift. |
| 8 | Permission seeding must run inside tenant context | Without `CurrentTenant.Change(tenantId)`, grants become host-level and don't apply to tenant users. |
| 9 | Angular proxy: re-run after every DTO change | Stale proxy = runtime type mismatches. |
| 10 | Use `DeleteBehavior.Restrict` on FKs by default | Prevents accidental cascade deletes. Only change if explicitly needed. |
| 11 | EnrichDtosAsync pattern for related names | Never N+1 — load all related records in one batch query, then assign by dictionary lookup. |
| 12 | Use `Clock.Now` never `DateTime.Now` | `Clock` is timezone-aware and mockable in tests. `DateTime.Now` breaks clock-dependent tests. |
| 13 | Never inject `DbContext` in application or domain services | Always go through `IRepository<T>`. Direct DbContext access creates coupling and bypasses ABP conventions. |
| 14 | Don't put business logic in AppService | Business logic belongs in entity domain methods or domain services. AppService orchestrates; it doesn't decide. |
| 15 | Generate GUID outside the entity | Use `GuidGenerator.Create()` in AppService and pass to entity constructor. Never call `Guid.NewGuid()` inside entity. |
| 16 | Repository only for aggregate roots | Child entities are managed through their parent aggregate. Never define `IRepository<ChildEntity>`. |
| 17 | Never `AddDefaultRepositories(includeAllEntities: true)` | This registers repositories for non-aggregate-root entities, which violates DDD and creates confusion. |
| 18 | Use `GetCancellationToken(cancellationToken)` in repos | Ensures the token is linked to ABP's request token, not just the passed-in parameter which may be `default`. |
| 19 | Don't mix Mapperly and AutoMapper | Pick one per project and use it consistently. Check existing `ApplicationAutoMapperProfile.cs` before choosing. |
| 20 | `FindTenantResultDto.tenantId` not `.id` | Using `.id` always returns `undefined` → `__tenant` missing from OIDC flow → tenant context lost. |

---

## Reference Files

- `references/angular-patterns.md` — Syncfusion grid, search/paging/sort, modal vs page form, export button
- `references/mvc-blazor.md` — MVC Razor Pages and Blazor CRUD patterns
- `references/advanced-patterns.md` — bulk operations, export/import, sub-collections, domain events
