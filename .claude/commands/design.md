Design a feature implementation and produce a step-by-step execution plan that `/implement` can consume directly.

## Input
- If $ARGUMENTS points to an .md file, read it as the feature specification.
- Otherwise treat $ARGUMENTS as a brief feature description.
- If neither is provided, ask the user what feature to design.

## Phase 1: Understand
1. Read the feature spec/description carefully.
2. Read the relevant `CLAUDE.md` files — these define conventions and current architecture:
   - `CLAUDE.md` (root) — project structure, tech stack, environments.
   - `src/backend/CLAUDE.md` — architecture, DynamoDB patterns, services, controllers.
   - `src/frontend/CLAUDE.md` — components, state, routing, Syncfusion.
   - `src/infra/CLAUDE.md` — CDK stacks, deployment, pipeline.
   - `e2e/CLAUDE.md` — Playwright conventions, page objects, fixtures, seed data, two-user model.
3. Check `docs/plans/` for existing feature specs (`*.spec.md`) and prior plans that may be relevant.
4. **Use parallel `Explore` agents** to analyze the layers affected by the feature — one agent per layer. Each agent should investigate the current state of that layer relevant to the feature (existing models, services, components, tables, etc.). Run them simultaneously to save time.
5. If the feature extends an existing feature, use a `feature-dev:code-explorer` agent to deep-trace the existing feature's execution path.
6. Use dedicated doc tools when the feature involves library APIs: `syncfusion-angular-assistant` MCP for Syncfusion components, `angular-cli` MCP for Angular APIs, `microsoft-docs:microsoft-code-reference` for .NET SDK/API verification, `awslabs.aws-documentation-mcp-server` for AWS services.
7. Codebase is the source of truth — if CLAUDE.md and code disagree, trust the code.
8. Identify which layers are affected: infrastructure, backend, frontend, e2e. Only include layers that actually need changes. E2E is in scope when the feature has user-facing behavior.

## Phase 2: Clarify
- If requirements are ambiguous or incomplete, ask the user targeted questions before proceeding.
- Do NOT guess or assume business rules — ask.

## Phase 3: Design
Use a `feature-dev:code-architect` agent to produce the architecture blueprint, providing it with the Explore agents' findings and the feature requirements.

Produce a design for each affected layer. Skip layers that don't need changes.

### Infrastructure (if needed)
- New/modified constructs in `MainStack`, DynamoDB tables, IAM roles, Lambda config, API Gateway routes.
- Environment variables to wire to Lambda via `api.Handler.AddEnvironment()`.
- Note any destructive changes that require careful rollout.

### Backend (if needed)
- New/modified entities, services, controllers, DTOs.
- DynamoDB access patterns (keys, indexes, query design).

### Frontend (if needed)
- New/modified components, services, routes, guards.
- UI layout and Syncfusion components to use.
- Auth considerations (stub + OIDC).

### E2E Tests (if feature has user-facing behavior)
Design E2E test scenarios for the feature. Reference the spec's "Use Cases (E2E Scenarios)" section and translate them into concrete Playwright tests:
- List each test scenario with: test name, user (Alice/Bob/both), steps, and expected assertions.
- Identify new page objects or extensions to existing page objects in `e2e/pages/`.
- Identify new `data-testid` attributes needed in Angular templates.
- Identify seed data changes needed in both `E2eSeeder.cs` and `seed-contracts.ts`.
- Follow conventions from `e2e/CLAUDE.md` (two-user model, `SEED[testUser]`, `authenticatedPage` fixture, etc.).

### Contracts (required when backend is in scope)
Define contracts for all affected layers in a structured format.

**REST API** (when backend + frontend are both in scope):

```
Endpoint: [METHOD] /api/[path]
Request:  [DTO name] { field: type, ... } (or empty for GET/DELETE)
Response: [DTO name] { field: type, ... }
Status:   200/201/204/400/404 — when each is returned
```

**Internal contracts** (always for backend changes):
- Service interfaces: `I{Name}Service` with method signatures.
- DynamoDB entity shapes and key schema.
- DTO definitions and AutoMapper mapping direction.

**Frontend types** (when frontend is in scope):
- Types/interfaces that map to backend DTOs.
- Mapper function signatures.

## Phase 4: Execution Plan
Create a numbered, ordered plan grouped by layer. For each step specify:
- **What**: file(s) to create or modify.
- **Why**: what this step achieves.
- **Details**: key implementation notes (not full code, just enough to be unambiguous).

Group steps per layer, with tests alongside implementation (matching how `/implement` executes):

1. **Infrastructure** (if needed)
   - Construct/resource changes → wiring → validation with `cdk diff`
   - Seed data changes (`E2eSeeder.cs`) if E2E tests require new baseline data
2. **Backend** (if needed)
   - Entities → Service interface + tests → Service implementation → Controller + DTOs + mapping → controller tests
3. **Frontend** (if needed)
   - Types/interfaces → Service + tests → Components/pages → Mapper + mapper tests → Routing
   - Add `data-testid` attributes to new/modified templates for E2E selectors
4. **E2E Tests** (if feature has user-facing behavior)
   - Update `seed-contracts.ts` to match any `E2eSeeder.cs` changes
   - New/extended page objects in `e2e/pages/`
   - Test specs in `e2e/tests/` — cover happy path, validation, and cross-user isolation

## Phase 5: Self-Review
Before presenting the plan, validate design-level concerns:
- [ ] Every requirement from the spec is addressed.
- [ ] No unnecessary changes — each step is justified by a requirement.
- [ ] No over-engineering — simplest approach that meets requirements.
- [ ] API contracts are consistent between backend and frontend.
- [ ] DynamoDB access patterns support the required queries without table scans.
- [ ] Destructive infra changes are flagged.
- [ ] The plan follows existing conventions from CLAUDE.md files — no pattern reinvention.
- [ ] E2E test scenarios cover user-facing behavior from the spec's Use Cases section.
- [ ] E2E design follows `e2e/CLAUDE.md` conventions (two-user model, page objects, fixtures, seed contracts).

If any check fails, revise the plan before presenting it.

## Output
1. Present the plan to the user for review.
2. **Wait for explicit approval** before saving. Ask: "Does this plan look good? Any changes before I save it?"
3. After approval, save to `docs/plans/{feature-name}.md` so `/implement` can reference it by file path.
4. For large features (multiple layers), remind the user they can implement one layer at a time: `/implement docs/plans/feature.md --layer backend`.
