# ABP CRUD: Angular UI Patterns

Use this file for Step 15. Select the pattern matching the spec decisions.

---

## Pattern A: Syncfusion Grid — Modal Create/Edit

Full-featured list with search, server-side paging, sorting, column resize, and modal dialogs.

### Component (TypeScript)

```typescript
// {feature}-list.component.ts
import { Component, OnInit, ViewChild } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule, ReactiveFormsModule, FormBuilder, FormGroup, Validators } from '@angular/forms';
import { GridModule, PageService, SortService, FilterService, ToolbarService,
         ResizeService, ExcelExportService, GridComponent } from '@syncfusion/ej2-angular-grids';
import { DialogModule, DialogComponent } from '@syncfusion/ej2-angular-popups';
import { ToasterService, ConfirmationService, Confirmation } from '@abp/ng.theme.shared';
import { {Entity}Service } from '@proxy/{entities}';
import { {Entity}Dto, Create{Entity}Dto, Update{Entity}Dto, Get{Entity}ListInput } from '@proxy/{entities}/models';

@Component({
  selector: 'app-{feature}-list',
  standalone: true,
  imports: [CommonModule, FormsModule, ReactiveFormsModule, GridModule, DialogModule],
  providers: [PageService, SortService, FilterService, ToolbarService, ResizeService, ExcelExportService],
  templateUrl: './{feature}-list.component.html',
})
export class {Feature}ListComponent implements OnInit {
  @ViewChild('grid') grid!: GridComponent;
  @ViewChild('formDialog') formDialog!: DialogComponent;

  items: {Entity}Dto[] = [];
  totalCount = 0;
  isLoading = false;

  // Search/filter state
  filterText = '';
  // relatedId filter (if applicable):
  // selected{Related}Id: string | null = null;
  // relatedItems: {Related}LookupDto[] = [];

  // Paging
  pageSettings = { pageSize: 20, pageSizes: [10, 20, 50, 100] };

  // Sort
  sortSettings = { columns: [{ field: 'creationTime', direction: 'Descending' as const }] };

  // Modal
  dialogHeader = 'New {Entity}';
  isEditMode = false;
  editingId: string | null = null;
  form!: FormGroup;
  dialogButtons = [
    { click: () => this.saveForm(), buttonModel: { content: 'Save', isPrimary: true } },
    { click: () => this.formDialog.hide(), buttonModel: { content: 'Cancel' } },
  ];

  constructor(
    private service: {Entity}Service,
    private fb: FormBuilder,
    private toaster: ToasterService,
    private confirmation: ConfirmationService,
  ) {}

  ngOnInit() {
    this.buildForm();
    this.load();
    // Load lookup data if needed:
    // this.{related}Service.getLookupList().subscribe(r => this.relatedItems = r);
  }

  buildForm() {
    this.form = this.fb.group({
      name: ['', [Validators.required, Validators.maxLength(256)]],
      // status: [{Entity}Status.Active, Validators.required],
      // {related}Id: [null],
      concurrencyStamp: [''],
    });
  }

  load() {
    this.isLoading = true;
    const input: Get{Entity}ListInput = {
      maxResultCount: this.pageSettings.pageSize,
      skipCount: 0,
      filter: this.filterText || undefined,
      // {related}Id: this.selected{Related}Id || undefined,
    };
    this.service.getList(input).subscribe({
      next: r => {
        this.items = r.items ?? [];
        this.totalCount = r.totalCount;
        this.isLoading = false;
      },
      error: () => { this.isLoading = false; },
    });
  }

  onSearch() { this.load(); }
  onClearSearch() { this.filterText = ''; this.load(); }

  // Grid paging event — use server-side paging
  onPageChange(args: any) {
    const input: Get{Entity}ListInput = {
      maxResultCount: args.pageSize,
      skipCount: (args.currentPage - 1) * args.pageSize,
      filter: this.filterText || undefined,
    };
    this.service.getList(input).subscribe(r => {
      this.items = r.items ?? [];
      this.totalCount = r.totalCount;
    });
  }

  openCreate() {
    this.isEditMode = false;
    this.editingId = null;
    this.dialogHeader = 'New {Entity}';
    this.form.reset({ /* default values */ });
    this.formDialog.show();
  }

  openEdit(item: {Entity}Dto) {
    this.isEditMode = true;
    this.editingId = item.id;
    this.dialogHeader = 'Edit {Entity}';
    this.form.patchValue({
      name: item.name,
      // status: item.status,
      // {related}Id: item.{related}Id,
      concurrencyStamp: item.concurrencyStamp,
    });
    this.formDialog.show();
  }

  saveForm() {
    if (this.form.invalid) { this.form.markAllAsTouched(); return; }
    const value = this.form.value;

    if (this.isEditMode && this.editingId) {
      const input: Update{Entity}Dto = {
        name: value.name,
        concurrencyStamp: value.concurrencyStamp,
      };
      this.service.update(this.editingId, input).subscribe({
        next: () => {
          this.toaster.success('Updated successfully');
          this.formDialog.hide();
          this.load();
        },
        error: (err) => {
          this.toaster.error(err?.error?.error?.message ?? 'An error occurred');
        },
      });
    } else {
      const input: Create{Entity}Dto = { name: value.name };
      this.service.create(input).subscribe({
        next: () => {
          this.toaster.success('Created successfully');
          this.formDialog.hide();
          this.load();
        },
        error: (err) => {
          this.toaster.error(err?.error?.error?.message ?? 'An error occurred');
        },
      });
    }
  }

  delete(item: {Entity}Dto) {
    // Use ABP's confirm dialog — NOT browser's native confirm()
    this.confirmation.warn(
      `Are you sure you want to delete "${item.name}"?`, 'Confirm'
    ).subscribe(status => {
      if (status === Confirmation.Status.confirm) {
        this.service.delete(item.id).subscribe(() => {
          this.toaster.success('Deleted successfully');
          this.load();
        });
      }
    });
  }

  exportExcel() {
    this.grid.excelExport({ fileName: '{entities}.xlsx' });
  }
}
```

### Template (HTML)

```html
<!-- {feature}-list.component.html -->
<div class="page-header">
  <h3>{Feature}</h3>
  <button ejs-button [isPrimary]="true" *abpPermission="'{App}.{Feature}.Create'"
          (click)="openCreate()">+ New {Entity}</button>
</div>

<!-- Search bar -->
<div class="filter-bar">
  <input type="text" placeholder="Search..." [(ngModel)]="filterText"
         (keyup.enter)="onSearch()" class="e-input" />
  <button ejs-button (click)="onSearch()">Search</button>
  <button ejs-button (click)="onClearSearch()">Clear</button>

  <!-- Dropdown filter (if related entity filter) -->
  <!-- <ejs-dropdownlist [dataSource]="relatedItems" [fields]="{text:'name',value:'id'}"
                    placeholder="All {Related}" [(ngModel)]="selected{Related}Id"
                    (change)="load()"></ejs-dropdownlist> -->

  <button ejs-button (click)="exportExcel()" *abpPermission="'{App}.{Feature}.Export'">
    Export Excel
  </button>
</div>

<!-- Grid -->
<ejs-grid #grid
  [dataSource]="items"
  [allowPaging]="true"
  [pageSettings]="pageSettings"
  [allowSorting]="true"
  [sortSettings]="sortSettings"
  [allowResizing]="true"
  [allowExcelExport]="true"
  (pageChange)="onPageChange($event)">
  <e-columns>
    <e-column field="name" headerText="Name" [allowSorting]="true"></e-column>
    <!-- <e-column field="{related}Name" headerText="{Related}"></e-column> -->
    <!-- <e-column field="status" headerText="Status"></e-column> -->
    <e-column field="creationTime" headerText="Created" type="date" format="yMd"></e-column>
    <e-column headerText="Actions" [allowSorting]="false" width="120">
      <ng-template #template let-data>
        <button ejs-button *abpPermission="'{App}.{Feature}.Update'"
                (click)="openEdit(data)">Edit</button>
        <button ejs-button cssClass="e-danger" *abpPermission="'{App}.{Feature}.Delete'"
                (click)="delete(data)">Delete</button>
      </ng-template>
    </e-column>
  </e-columns>
</ejs-grid>

<!-- Total count -->
<small>{{ totalCount }} record(s)</small>

<!-- Create/Edit Modal Dialog -->
<ejs-dialog #formDialog
  [header]="dialogHeader"
  [isModal]="true"
  [visible]="false"
  width="500px"
  [buttons]="dialogButtons">
  <ng-template #content>
    <form [formGroup]="form">
      <div class="form-group">
        <label>Name *</label>
        <input ejs-textbox formControlName="name" placeholder="Name" />
        <div *ngIf="form.get('name')?.invalid && form.get('name')?.touched" class="error">
          Name is required (max 256 chars)
        </div>
      </div>
      <!-- Add more fields here per spec -->
    </form>
  </ng-template>
</ejs-dialog>
```

---

## Pattern B: Separate Page Form (instead of modal)

Use this when the create/edit form is complex (many fields, nested sections, file uploads).

### File structure
```
{feature}/
├── {feature}-list.component.ts     (same as Pattern A but no modal; navigate to form instead)
├── {feature}-list.component.html
├── {feature}-form.component.ts     (create + edit combined — reads :id param)
└── {feature}-form.component.html
```

### Form component

```typescript
// {feature}-form.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { {Entity}Service } from '@proxy/{entities}';

@Component({
  selector: 'app-{feature}-form',
  standalone: true,
  imports: [ReactiveFormsModule, /* other imports */],
  templateUrl: './{feature}-form.component.html',
})
export class {Feature}FormComponent implements OnInit {
  form!: FormGroup;
  isEditMode = false;
  entityId: string | null = null;
  isLoading = false;

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private service: {Entity}Service,
    private fb: FormBuilder,
  ) {}

  ngOnInit() {
    this.entityId = this.route.snapshot.paramMap.get('id');
    this.isEditMode = !!this.entityId;
    this.buildForm();
    if (this.isEditMode) this.loadForEdit();
  }

  buildForm() {
    this.form = this.fb.group({
      name: ['', [Validators.required, Validators.maxLength(256)]],
      concurrencyStamp: [''],
    });
  }

  loadForEdit() {
    this.isLoading = true;
    this.service.get(this.entityId!).subscribe(entity => {
      this.form.patchValue(entity);
      this.isLoading = false;
    });
  }

  save() {
    if (this.form.invalid) { this.form.markAllAsTouched(); return; }
    const value = this.form.value;

    const obs = this.isEditMode
      ? this.service.update(this.entityId!, value)
      : this.service.create(value);

    obs.subscribe(() => this.router.navigate(['/{feature}']));
  }

  cancel() { this.router.navigate(['/{feature}']); }
}
```

### List component (navigates instead of opening modal)

```typescript
// Replace openCreate() and openEdit() in Pattern A:
openCreate() { this.router.navigate(['/{feature}/new']); }
openEdit(item: {Entity}Dto) { this.router.navigate(['/{feature}', item.id, 'edit']); }
```

---

## Pattern C: Syncfusion TreeGrid (hierarchical/parent-child entities)

Use when the entity has a parent-child self-reference (e.g. tasks with sub-tasks, org units).

```typescript
import { TreeGridModule, EditService, ToolbarService,
         FilterService, SortService, PageService } from '@syncfusion/ej2-angular-treegrid';

// Key differences from regular Grid:
// - dataSource must be flat with parentIdMapping
// - idMapping and parentIdMapping are required columns
// - isPrimaryKey must be set on the id column

// taskSettings config:
taskSettings = {
  idMapping: 'id',           // field that holds the row ID
  parentID: 'parentId',      // field that holds parent row ID (null = root)
  hasChildMapping: 'hasChildren',  // optional: tells grid which rows have children
};
```

HTML:
```html
<ejs-treegrid [dataSource]="items" childMapping="children"  <!-- or use taskSettings for flat -->
  [allowPaging]="true" [allowSorting]="true" [allowResizing]="true">
  <e-columns>
    <e-column field="id" [isPrimaryKey]="true" [visible]="false"></e-column>
    <e-column field="name" headerText="Name"></e-column>
    <!-- other columns -->
  </e-columns>
</ejs-treegrid>
```

---

## Permission Guards

Always guard buttons by permission:

```html
<!-- Template directive -->
<button *abpPermission="'{App}.{Feature}.Create'">New</button>

<!-- Multiple permissions (AND) -->
<button *abpPermission="'{App}.{Feature}.Update && SomeOther.Permission'">Edit</button>
```

In TypeScript (programmatic check):
```typescript
import { PermissionService } from '@abp/ng.core';

constructor(private permissionService: PermissionService) {}

canEdit(): boolean {
  return this.permissionService.getGrantedPolicy('{App}.{Feature}.Update');
}
```

---

## Error Handling (toast notifications)

Use ABP's built-in notification service for user feedback:

```typescript
import { ToasterService } from '@abp/ng.theme.shared';

constructor(private toaster: ToasterService) {}

// On success:
this.service.create(input).subscribe({
  next: () => {
    this.toaster.success('Created successfully');
    this.load();
  },
  error: (err) => {
    this.toaster.error(err?.error?.error?.message ?? 'An error occurred');
  }
});
```
