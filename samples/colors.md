# Implementation Plan: Scope of Work — Color Improvements

**Spec:** `docs/plans/admin/scope-of-work-color.spec.md`

## Contracts

**No API/DTO changes** — the existing `ScopeOfWorkDto` and `ScopeOfWork` entity already have a `Color` field. The backend will silently override the client-sent color for Tasks.

**Service interface** — no new methods. `CreateAsync` and `UpdateAsync` gain internal logic but signatures stay the same.

---

## Step 1 — Backend: Update `ScopeOfWorkService.CreateAsync` for Task color inheritance

**File:** `src/backend/RenoHouse.BusinessLayer/ScopeOfWork/ScopeOfWorkService.cs`

**What:** When creating a Task (ParentId ≠ "ROOT"), query the parent Trade via `GetByIdAsync` and set `item.Color` to the Trade's Color. If the parent is not found, throw `BusinessValidationException`.

**Details:**
- After the Level assignment line (`item.Level = ...`), add Task color inheritance:
  ```csharp
  if (item.Level == ScopeOfWorkLevel.Task)
  {
      var parentTrade = await GetByIdAsync(item.ParentId, cancellationToken)
          ?? throw new BusinessValidationException($"Parent Trade with Id '{item.ParentId}' not found.");
      item.Color = parentTrade.Color;
  }
  ```
- Keep the existing default-color fallback (`#FFFFFF`) but restrict it to Trades only — wrap in `else if` or move inside an `item.Level == Trade` check.

---

## Step 2 — Backend: Update `ScopeOfWorkService.UpdateAsync` for cascade and Task override

**File:** `src/backend/Project.BusinessLayer/ScopeOfWork/ScopeOfWorkService.cs`

**What:** Two behaviors:
1. **Task update:** Set `item.Color` to the parent Trade's current color, ignoring the client value.
2. **Trade update with color change:** After saving the Trade, if `item.Color != existing.Color`, query all child Tasks via `ParentIdIndex` and update each Task's `Color`.

**Details:**
- After the existing `GetByIdAsync` check and Level assignment, add Task color override:
  ```csharp
  if (item.Level == ScopeOfWorkLevel.Task)
  {
      var parentTrade = await GetByIdAsync(item.ParentId, cancellationToken)
          ?? throw new BusinessValidationException($"Parent Trade with Id '{item.ParentId}' not found.");
      item.Color = parentTrade.Color;
  }
  ```
- Capture `existing.Color` before `SaveAsync(item)`. After saving, add Trade color cascade:
  ```csharp
  if (item.Level == ScopeOfWorkLevel.Trade && item.Color != existing.Color)
  {
      var childSearch = _dynamoDbContext.QueryAsync<ScopeOfWorkEntity>(
          item.Id, new QueryConfig { IndexName = "ParentIdIndex" });
      var children = await childSearch.GetRemainingAsync(cancellationToken);
      foreach (var child in children)
      {
          child.Color = item.Color;
          await _dynamoDbContext.SaveAsync(child, cancellationToken);
      }
  }
  ```

---

## Step 3 — Backend: Add unit tests for color behavior

**File:** `src/backend/RenoHouse.Tests/ScopeOfWork/ScopeOfWorkServiceTests.cs`

**What:** Add tests in the existing `CreateAsync Tests` and `UpdateAsync Tests` regions.

### CreateAsync Tests to add:
- `CreateAsync_ForTask_InheritsColorFromParentTrade` — Create a Task with ParentId pointing to a Trade with color `#FF5733`. Assert the returned item's Color is `#FF5733`, regardless of the input color. Use `SetupGetByIdReturns` for the parent Trade.
- `CreateAsync_ForTask_WhenParentNotFound_ThrowsBusinessValidationException` — Use `SetupGetByIdReturnsNull` for the parent. Assert `BusinessValidationException`.

### UpdateAsync Tests to add:
- `UpdateAsync_TradeColorChange_CascadesToChildTasks` — Set up a Trade with `#FF0000`, two child Tasks. Update Trade color to `#00FF00`. Mock `ParentIdIndex` query to return the children (same pattern as `DeleteAsync_WhenTrade_DeletesChildrenFirst`). Verify `SaveAsync` was called for each child with the new color.
- `UpdateAsync_TradeColorUnchanged_DoesNotCascade` — Update Trade with same color. Verify no `ParentIdIndex` query occurs.
- `UpdateAsync_Task_SetsColorFromParentTrade` — Set up a parent Trade with `#3498DB`. Update a Task sending `#FF0000`. Verify saved Task has `#3498DB`.

---

## Step 4 — Frontend: Add web-safe 216 palette constant

**File:** `src/frontend/src/app/features/scope-of-work/scope-of-work.page.ts`

**What:** Add a `presetColors` property containing the web-safe 216 color palette as a class property.

**Details:**
```typescript
readonly webSafePalette: { [key: string]: string[] } = {
  'custom': [/* all 216 web-safe colors: R,G,B ∈ {00,33,66,99,CC,FF} as #RRGGBB */]
};
```

The 216 colors are all combinations of R, G, B where each channel is one of: `00`, `33`, `66`, `99`, `CC`, `FF`.

---

## Step 5 — Frontend: Update template for palette picker, hide Task color, row selection

**File:** `src/frontend/src/app/features/scope-of-work/scope-of-work.page.html`

### 5a. Trades grid — row-only selection
Change `[selectionSettings]="{ type: 'Single' }"` to `[selectionSettings]="{ type: 'Single', mode: 'Row' }"`.

### 5b. Tasks grid — remove Color column
Delete the `<e-column headerText="Color" ...>` block (with its `ng-template`) from the tasks `ejs-grid`.

### 5c. Edit dialog — hide color picker for Tasks
Wrap the color `form-field` div with `@if (editingParentId() === 'ROOT')`.

### 5d. Edit dialog — palette-only color picker
Update the `ejs-colorpicker` attributes:
- Add `mode="Palette"`
- Add `[columns]="18"`
- Add `[presetColors]="webSafePalette"`
- Change `[modeSwitcher]="true"` → `[modeSwitcher]="false"`

Result:
```html
@if (editingParentId() === 'ROOT') {
  <div class="form-field">
    <label class="form-label">Color</label>
    <input ejs-colorpicker type="color" formControlName="color"
           mode="Palette" [columns]="18"
           [presetColors]="webSafePalette"
           [modeSwitcher]="false" [showButtons]="true" />
  </div>
}
```

---

## Step 6 — Frontend: Expose `editingParentId` for template

**File:** `src/frontend/src/app/features/scope-of-work/scope-of-work.page.ts`

**What:** Change `private readonly editingParentId` to `protected readonly editingParentId` so the template can use it in the `@if` condition.

---

## Step 7 — E2E: Update seed data

**Files:**
- `src/infra/RenoHouse.Infra/E2eSeeder.cs`
- `e2e/test-data/seed-contracts.ts`

**What:** Change Task colors to match their parent Trade's color:

| Task | Old Color | New Color | Parent Trade |
|------|-----------|-----------|-------------|
| e2e-sow-101 (Install water heater) | `#FFFFFF` | `#FF5733` | Plumbing |
| e2e-sow-102 (Replace pipes) | `#FFFFFF` | `#FF5733` | Plumbing |
| e2e-sow-103 (Wire outlets) | `#FFFFFF` | `#3498DB` | Electrical |

Update both files to keep them in sync.

---

## Step 8 — E2E: Update page object column comments

**File:** `e2e/pages/scope-of-work.page.ts`

**What:** Update column index comments in `taskNames()` and `taskFormula()`:
```
// Old: Columns: #(0), Acronym(1), Name(2), DurationFormula(3), Color(4), Actions(5)
// New: Columns: #(0), Acronym(1), Name(2), DurationFormula(3), Actions(4)
```

No actual index changes — Name (2) and DurationFormula (3) stay at the same positions. Action buttons use `[title]` selectors, not indices.

---

## Step 9 — E2E: Add color-specific E2E tests

**File:** `e2e/tests/scope-of-work.spec.ts`

### Tests to add:

1. **`task edit dialog has no color picker`**
   - Open edit dialog for seeded Task "Install water heater" under "Plumbing"
   - Assert no `ejs-colorpicker` / color input is visible in the dialog
   - Close dialog

2. **`tasks grid has no Color column`**
   - Navigate to SoW page, select "Plumbing" Trade
   - Assert the tasks grid header row does not contain "Color" text

3. **`new task inherits trade color via API`**
   - Create a Trade with a known color, create a Task under it
   - Fetch all ScopeOfWork items via GET `/api/ScopeOfWork`
   - Assert the Task's color matches the Trade's color

4. **`trade color change cascades to tasks via API`**
   - Use a seeded Trade, update its color via PUT `/api/ScopeOfWork`
   - Fetch all items, verify child Tasks have the new color
   - (Restore original color afterward to avoid test pollution)

Tests 3 and 4 verify backend behavior via API since Task color is not visible in the Tasks grid UI. Use the `apiToken` fixture from `e2e/fixtures/api.fixture.ts` for direct API calls.

---

## Execution Order

| # | Layer | Steps | Notes |
|---|-------|-------|-------|
| 1 | Backend | Steps 1–3 | Service logic + tests |
| 2 | Frontend | Steps 4–6 | Palette, template, visibility |
| 3 | E2E | Steps 7–9 | Seed data, page object, tests |

Each layer can be implemented and verified independently. Run `dotnet test` after backend steps, `npm test` after frontend steps.
