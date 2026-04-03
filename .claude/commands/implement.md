Implement a feature according to the provided plan.

Prerequisite: the plan should already define scope (which layers), contracts (REST endpoints, interfaces, DTOs), steps, and E2E test design (when the feature has user-facing behavior). If the plan is missing these, stop and ask the user to run `/design` first.

## Input
- If $ARGUMENTS points to a file, read it as the implementation plan.
- If $ARGUMENTS contains `--layer infra|backend|frontend|e2e`, implement only that layer from the plan. This is useful for large features or when continuing across conversations.
- Otherwise treat $ARGUMENTS as a reference to a plan discussed in conversation.
- If no plan is identifiable, stop and ask the user.

## Before You Start
1. Read the plan and identify affected layers: infrastructure, backend, frontend, e2e.
2. Read the relevant `CLAUDE.md` files for each affected layer — these are the source of truth for conventions:
   - `src/infra/CLAUDE.md` — CDK patterns, deployment, pipeline.
   - `src/backend/CLAUDE.md` — architecture, DynamoDB, services, controllers, testing.
   - `src/frontend/CLAUDE.md` — components, state, routing, Syncfusion, testing.
   - `e2e/CLAUDE.md` — Playwright conventions, page objects, fixtures, seed data, two-user model.
3. Validate that the plan includes:
   - Contracts (REST endpoints, interfaces, DTOs) — required when backend is in scope. If missing, define them before writing implementation code.
   - E2E test design (test scenarios, page objects, selectors) — required when e2e is in scope. If missing, stop and ask the user to run `/design` to add it.

## Implementation Loop
Execute one layer at a time in order: **infrastructure → backend → frontend → e2e**. Skip layers not in scope.

For **infrastructure, backend, and frontend** layers, follow the red/green/refactor loop below. For **e2e**, follow the E2E-specific section instead (E2E tests cannot run locally).

### Step 1: Write tests first
- Write tests that verify the contracts and expected behavior.
- Run tests — they MUST fail (red phase). If they pass, the test is not testing anything new.

### Step 2: Implement
- Write the minimum code to make tests pass.
- Follow conventions from the layer's `CLAUDE.md` — do not reinvent patterns.
- When using unfamiliar or recently updated library APIs, use dedicated doc tools: `syncfusion-angular-assistant` MCP for Syncfusion, `angular-cli` MCP for Angular, `microsoft-docs:microsoft-code-reference` for .NET SDK verification.
- After each significant change:
  1. Predict which tests should now pass or break.
  2. Run tests.
  3. Compare predicted vs actual results.
  4. If mismatch: find root cause and fix before moving on.

### Step 3: Layer checkpoint
- All new tests for this layer pass.
- No previously passing tests are broken.
- Use a `feature-dev:code-reviewer` agent to review the layer's changes before moving on. Provide it with the plan context so it can check against intended design.
- If the reviewer finds issues, fix them before proceeding.
- Inform the user of progress before moving to the next layer.

### Frontend-specific
- **The red/green/refactor loop applies to frontend the same as backend. Do NOT skip it.**
- Write Vitest unit tests BEFORE implementation for:
  - **Services**: HTTP endpoint tests (method, URL, request body) — follow `real-estate.service.spec.ts` pattern with `HttpTestingController`. Type alignment tests verifying the interface matches backend DTO fields.
  - **Guards**: Test that the guard returns `true` for authorized users and a `UrlTree` redirect for unauthorized users. Mock `AuthService`.
  - **Component logic**: Any non-trivial logic in page components — ordering, validation, filtering, computed signals. Extract testable logic into methods and test them. If logic is too coupled to the template, at minimum test via `TestBed` with the component.
- Run `npm test` after writing tests to confirm they FAIL (red phase), then implement to make them pass (green phase).
- When the plan includes significant UI work (new pages, complex layouts), use the `frontend-design` skill to generate polished UI components.
- **Tabular data pages: use Syncfusion `ejs-grid`, not custom `@for` lists.** Follow the RealEstate page pattern (`real-estate.page.ts/html/css`):
  - Component: `GridModule` import, `[dataSource]="items()"` bound to a computed signal, `(recordClick)="onRowClick($event)"` for edit-on-click, action column with `<ng-template #template>` for delete buttons.
  - Template: `<ejs-grid>` with `<e-columns>`, `gridLines="Horizontal"`, `cssClass="rh-clickable-grid"`. Custom cell rendering via `<ng-template #template let-data>` for non-trivial display (e.g., enum labels).
  - CSS: Copy the grid override block from `real-estate.page.css` (`:host ::ng-deep .e-grid` styles) for consistent look.
  - Loading: Use `effect()` with `onCleanup` to tie overlay to `isLoading()` signal — same as RealEstate, NOT manual `overlay.show()/hide()` in methods.
  - Dialog: Same pattern — `viewChild.required<DialogComponent>`, `editDialog().show()` in TS, `editDialog.hide()` in template (template ref).
  - E2E: Page object uses `grid.locator('tr.e-row')` for rows, `td` column selectors. No assertions in page object — keep those in spec files. Rely on Playwright's auto-retrying `toHaveCount()` after mutations instead of explicit DOM waits.
- Add `data-testid` attributes to new/modified templates per the plan's E2E test design — these are needed for E2E selectors.
- When adding new pages or significant user-facing features, add PostHog event tracking via `PosthogService` following existing patterns (see `posthog.service.ts`). Use `posthog:posthog-instrumentation` skill if unsure about instrumentation patterns.
- **Error handling in catch blocks — never swallow errors silently.** Every `catch` block that handles a user-initiated action (save, delete, load) MUST provide all three layers of feedback:
  1. `console.error(...)` — for developer debugging.
  2. `this.posthog.captureError(...)` — for production monitoring.
  3. `this.toast.showError(...)` — for user-facing notification (inject `ToastService` from `core/services/toast.service`).
  Bare `catch {}` or `catch { console.error() }` without user feedback is not acceptable. The only exception is auth-infrastructure code (e.g., JWT parsing in guards/services) where a toast would confuse users — there, use `console.warn` + `posthog.captureError` instead.

### Infrastructure-specific
- If the plan includes seed data changes, update `E2eSeeder.cs` during the infra layer (it lives in `src/infra/`).

### E2E-specific (does NOT follow the red/green/refactor loop above)
- E2E tests run against the deployed `e2e` environment, not locally. They cannot be run as part of the red/green/refactor loop. Instead, write the tests so they are ready to run after deployment via `deploy-and-test.sh`.
- Follow the implementation order from the plan:
  1. Verify `seed-contracts.ts` matches `E2eSeeder.cs` (already updated in the infra layer if seed data changed).
  2. Create/extend page objects in `e2e/pages/` — one class per page, locators use `data-testid` attributes.
  3. Write test specs in `e2e/tests/` — import `test` and `expect` from `../fixtures/app.fixture`.
  4. Use `authenticatedPage` fixture for authenticated tests, `testUser` + `SEED[testUser]` for data assertions.
  5. Use `Date.now()` suffixes for destructive test data to avoid cross-browser collisions.
  6. Never delete seeded baseline items — only delete items created during the test.
- **Syncfusion popup locators — avoid `.e-popup`.** The `.e-popup` class is shared by dialogs, dropdowns, comboboxes, and other Syncfusion components. When a dialog is open and a dropdown opens inside it, `page.locator('.e-popup')` matches the dialog wrapper (already visible) instead of the dropdown. Use specific selectors:
  - Dropdown/ComboBox popup: `.e-list-parent` (contains `.e-list-item` elements)
  - Dialog: `ejs-dialog` (the Angular component selector)
  - Confirm dialog OK button: `[data-testid="confirm-dialog-ok"]`
- **Overlay handling — use `waitForOverlayToClose()` from `helpers.ts`, not custom waits.** This helper waits for `networkidle` + overlay-backdrop hidden + dismisses toasts. Follow the pattern:
  - `goto()`: navigate → `waitForOverlayToClose()` → `root.waitFor({ state: 'visible' })`
  - `saveForm()`: click save button → `dialog.waitFor({ state: 'hidden' })` → `waitForOverlayToClose()`
  - `deleteItem()`: click delete → click confirm → `waitForOverlayToClose()`
  - After mutations, rely on Playwright auto-retrying `toHaveCount()` in the **spec** (not page object) to wait for the grid/list to update. Do NOT add explicit DOM waits or assertions inside page objects.
- Use a `feature-dev:code-reviewer` agent to review new E2E tests against `e2e/CLAUDE.md` conventions before marking this layer complete.

## Cross-Layer Wiring
When multiple layers are involved, ensure they connect correctly:
- **Infra → Backend:** new resources wired to Lambda via `api.Handler.AddEnvironment()` in `MainStack.cs`.
- **Backend → Frontend:** REST contracts match — endpoint paths, HTTP methods, request/response shapes are consistent.
- **Auth:** new features work in both stub and OIDC modes.
- **Seed data:** `E2eSeeder.cs` (C#) and `seed-contracts.ts` (TypeScript) must define identical data.
- **Frontend → E2E:** every `data-testid` used in page objects exists in the Angular templates.

## When Done
Inform the user that implementation is complete and suggest running `/verify` with the spec file for full traceability:
```
/verify docs/plans/{feature-name}.spec.md
```
