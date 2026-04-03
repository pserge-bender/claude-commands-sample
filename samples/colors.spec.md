# Scope of Work — Color Improvements

## Goal
Simplify color management in the Scope of Work admin page by making Task colors automatically inherit from their parent Trade. Replace the freeform color picker with a palette-only picker using the standard web-safe 216 colors, and improve grid UX by enforcing row-level selection.

## Scope

| Layer | Changes |
|-------|---------|
| **Backend** | Auto-set Task color on create; cascade Trade color to child Tasks on update |
| **Frontend** | Remove color column/picker from Tasks; replace color picker with palette mode; set row-only grid selection |
| **E2E** | Update seed data; update test assertions for color behavior |
| **Infrastructure** | None |

## Requirements

1. **Backend — Task color on create:** When `CreateAsync` is called for a Task (ParentId ≠ "ROOT"), the service must query the parent Trade and set the Task's `Color` to the Trade's `Color`, ignoring any color value sent by the client.

2. **Backend — Cascade color on Trade update:** When `UpdateAsync` is called for a Trade (ParentId = "ROOT") and the color has changed compared to the existing record, the service must query all child Tasks via `ParentIdIndex` and update each Task's `Color` to the new Trade color.

3. **Backend — Task color on update (ignore client value):** When `UpdateAsync` is called for a Task, the service must set `Color` to the parent Trade's current color, ignoring any color value sent by the client. This ensures lazy correction of legacy data.

4. **Frontend — Remove color from Tasks grid:** Remove the "Color" column from the Tasks `ejs-grid`.

5. **Frontend — Remove color picker from Task edit dialog:** When the edit dialog is opened for a Task (parentId ≠ "ROOT"), hide the color form field. The `color` form control can still exist but should not be visible or editable for Tasks.

6. **Frontend — Palette-only color picker for Trades:** Replace the current `ejs-colorpicker` with palette mode using the standard web-safe 216 color palette. Disable the mode switcher (`[modeSwitcher]="false"`), so users can only pick from the palette — no freeform input.

7. **Frontend — Row-only selection on Trades grid:** Set `selectionSettings` on the Trades grid to `{ type: 'Single', mode: 'Row' }` to prevent cell-level selection highlighting. (The Tasks grid can remain as-is since clicking a task row opens the edit dialog.)

8. **E2E seed data:** Update `seed-contracts.ts` and `E2eSeeder.cs` so seeded Tasks carry their parent Trade's color instead of `#FFFFFF`. Specifically: Plumbing tasks get `#FF5733`, Electrical tasks get `#3498DB`.

9. **Backend unit tests:** Add tests for: (a) Task creation inherits Trade color, (b) Trade color update cascades to child Tasks, (c) Task update gets Trade color regardless of client value.

## Use Cases (E2E Scenarios)

- **As an admin, when I create a new Trade with a color, then the color swatch appears in the Trades grid.**
- **As an admin, when I create a new Task under a Trade, then the Task inherits the Trade's color (visible via API response, not in the Tasks grid UI).**
- **As an admin, when I edit a Trade and change its color, then all child Tasks receive the updated color (verifiable via API GET).**
- **As an admin, when I edit a Task, I do not see a color picker in the dialog.**
- **As an admin, when I view the Tasks grid, there is no Color column.**
- **As an admin, when I click a Trades grid row, the entire row is selected (no individual cell highlighting).**

## Constraints

- No one-time data migration. Existing Tasks with stale colors will be corrected lazily when their parent Trade is updated or when the Task itself is updated.
- The web-safe 216 color palette is the fixed set — no custom additions.
- No changes to the ScopeOfWork DynamoDB schema — the `Color` field already exists on all items.
- No changes to the DTO or API contract — `Color` still flows in both directions; the backend simply overrides the client value for Tasks.

## Assumptions

- The standard web-safe 216 color palette (6×6×6 RGB cube: `#000000` to `#FFFFFF` in `00/33/66/99/CC/FF` steps) is appropriate for this use case.
- Syncfusion's `ColorPicker` in `mode="Palette"` with `presetColors` supports the 216-color grid natively.
- The cascade update on Trade color change is acceptable as a synchronous operation within the same request, since the number of Tasks per Trade is small (typically < 20).
- Row-only selection is only needed on the Trades grid; the Tasks grid behavior (click-to-edit) remains unchanged.

## Open Questions

None — all questions resolved during clarification.
