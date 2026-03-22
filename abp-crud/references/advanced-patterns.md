# ABP CRUD: Advanced Patterns

## Integration Tests

If tests were requested in spec question 18:

**New file:** `test/{App}.Application.Tests/{Feature}/{Entity}AppServiceTests.cs`

```csharp
using Shouldly;
using Volo.Abp.Modularity;
using Xunit;

namespace {RootNamespace}.{Feature};

public abstract class {Entity}AppServiceTests<TStartupModule> : {App}ApplicationTestBase<TStartupModule>
    where TStartupModule : IAbpModule
{
    private readonly I{Entity}AppService _service;

    protected {Entity}AppServiceTests()
    {
        _service = GetRequiredService<I{Entity}AppService>();
    }

    [Fact]
    public async Task Should_Create_{Entity}()
    {
        var input = new Create{Entity}Dto { Name = "Test {Entity}" };
        var result = await _service.CreateAsync(input);

        result.ShouldNotBeNull();
        result.Name.ShouldBe("Test {Entity}");
        result.Id.ShouldNotBe(Guid.Empty);
    }

    [Fact]
    public async Task Should_Get_List()
    {
        var result = await _service.GetListAsync(new Get{Entity}ListInput());
        result.TotalCount.ShouldBeGreaterThanOrEqualTo(0);
        result.Items.ShouldNotBeNull();
    }

    [Fact]
    public async Task Should_Update_{Entity}()
    {
        // Create first
        var created = await _service.CreateAsync(new Create{Entity}Dto { Name = "Original" });

        // Update
        var updated = await _service.UpdateAsync(created.Id, new Update{Entity}Dto
        {
            Name = "Updated",
            ConcurrencyStamp = created.ConcurrencyStamp,
        });

        updated.Name.ShouldBe("Updated");
    }

    [Fact]
    public async Task Should_Delete_{Entity}()
    {
        var created = await _service.CreateAsync(new Create{Entity}Dto { Name = "To Delete" });
        await _service.DeleteAsync(created.Id);
        await Should.ThrowAsync<EntityNotFoundException>(() => _service.GetAsync(created.Id));
    }
}
```

**Notes:**
- Uses real SQLite in-memory DB — no mocking.
- `Shouldly` for assertions: `.ShouldBe()`, `.ShouldNotBeNull()`, `.ShouldThrowAsync<T>()`.
- Test naming: `Should_[ExpectedBehavior]_When_[Condition]`.
- Add `context.Services.AddAlwaysAllowAuthorization()` in test module to bypass permission checks.

---

## Bulk Operations

### AppService
```csharp
[Authorize({App}Permissions.{Feature}.Delete)]
public async Task<BulkResultDto> BulkDeleteAsync(List<Guid> ids)
{
    var result = new BulkResultDto();
    foreach (var id in ids)
    {
        try { await _repository.DeleteAsync(id); result.SuccessCount++; }
        catch (Exception ex) { result.FailedCount++; result.Errors.Add(ex.Message); }
    }
    return result;
}
```

### Controller
```csharp
[HttpPost("bulk-delete")]
public Task<BulkResultDto> BulkDeleteAsync([FromBody] List<Guid> ids)
    => _appService.BulkDeleteAsync(ids);
```

---

## Sub-Collections (e.g. Comments, Attachments)

### Entity Relationship
```csharp
// In the main entity
public ICollection<{Entity}Comment> Comments { get; set; } = new List<{Entity}Comment>();
```

### Repository: Include Navigation
```csharp
return await (await GetDbSetAsync())
    .Include(x => x.Comments)
    .Where(x => x.Id == id)
    .FirstOrDefaultAsync(cancellationToken);
```

### AppService Methods
```csharp
public async Task<List<{Entity}CommentDto>> GetCommentsAsync(Guid entityId)
{
    var entity = await _repository.GetAsync(entityId);  // or query _commentRepository directly
    return ObjectMapper.Map<List<{Entity}Comment>, List<{Entity}CommentDto>>(entity.Comments.ToList());
}

public async Task<{Entity}CommentDto> AddCommentAsync(Guid entityId, string content)
{
    var comment = new {Entity}Comment(GuidGenerator.Create(), entityId, content);
    await _commentRepository.InsertAsync(comment, autoSave: true);
    return ObjectMapper.Map<{Entity}Comment, {Entity}CommentDto>(comment);
}
```

### Controller Routes
```csharp
[HttpGet("{entityId}/comments")]
public Task<List<{Entity}CommentDto>> GetCommentsAsync(Guid entityId)
    => _appService.GetCommentsAsync(entityId);

[HttpPost("{entityId}/comments")]
public Task<{Entity}CommentDto> AddCommentAsync(Guid entityId, [FromBody] string content)
    => _appService.AddCommentAsync(entityId, content);
```

---

## Domain Events

Raise events from the entity for cross-aggregate side effects:
```csharp
// In entity constructor or method
AddLocalEvent(new {Entity}CreatedEventData { {Entity}Id = Id });

// Event data class (Domain.Shared)
public class {Entity}CreatedEventData
{
    public Guid {Entity}Id { get; set; }
}

// Handler (Application or Domain layer)
public class {Entity}CreatedEventHandler
    : ILocalEventHandler<{Entity}CreatedEventData>,
      ITransientDependency
{
    public async Task HandleEventAsync({Entity}CreatedEventData data)
    {
        // side effects: send notification, update stats, etc.
    }
}
```

---

## Export (CSV/Excel)

```csharp
public interface I{Entity}AppService : IApplicationService
{
    Task<IRemoteStreamContent> ExportAsync(string format = "csv");
}

[Authorize({App}Permissions.{Feature}.Export)]
public async Task<IRemoteStreamContent> ExportAsync(string format = "csv")
{
    var items = await _repository.GetFilteredListAsync(maxResultCount: int.MaxValue);
    var dtos = ObjectMapper.Map<List<{Entity}>, List<{Entity}Dto>>(items);

    if (format == "csv")
    {
        var csv = BuildCsv(dtos);
        var bytes = System.Text.Encoding.UTF8.GetBytes(csv);
        return new RemoteStreamContent(new MemoryStream(bytes), "{entities}.csv", "text/csv");
    }
    throw new UserFriendlyException($"Unsupported format: {format}");
}
```

---

## Batch Enrichment (avoid N+1 queries)

When DTOs need data from related entities, load all related records in one query:
```csharp
private async Task EnrichDtosAsync(List<{Entity}Dto> dtos)
{
    // Collect all foreign key IDs
    var relatedIds = dtos
        .Where(d => d.RelatedEntityId.HasValue)
        .Select(d => d.RelatedEntityId!.Value)
        .Distinct()
        .ToList();

    if (!relatedIds.Any()) return;

    // Single batch query
    var related = await _relatedRepository.GetListAsync(x => relatedIds.Contains(x.Id));
    var lookup = related.ToDictionary(x => x.Id);

    // Assign to DTOs
    foreach (var dto in dtos)
        if (dto.RelatedEntityId.HasValue && lookup.TryGetValue(dto.RelatedEntityId.Value, out var rel))
            dto.RelatedEntityName = rel.Name;
}
```

---

## Concurrency Stamp on Frontend (Angular)

When opening an edit form, include `concurrencyStamp` from the loaded DTO:
```typescript
openEdit(entity: EntityDto) {
    this.editForm.patchValue({
        ...entity,
        concurrencyStamp: entity.concurrencyStamp  // must be sent back on update
    });
}
```

---

## Permission Seeding in Tenant Context

Always run permission seeding within the correct tenant context:
```csharp
using (_currentTenant.Change(tenantId))
{
    await SeedPermissionsAsync();
}
```

Without this, grants end up with `TenantId = NULL` (host-level) and won't apply to tenant users.
