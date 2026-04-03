Review the codebase and update project documentation to match the current state of the code.

Codebase is the source of truth. Documentation follows code, not the other way around.

## Input
- If $ARGUMENTS specifies a scope (e.g., `backend`, `frontend`, `infra`, `e2e`, `all`), focus on that area.
- If $ARGUMENTS is empty, review all areas.

## Phase 1: Parallel Analysis
Launch **parallel `Explore` agents** — one per area — to compare documentation against code:

1. **Backend agent:** Compare `src/backend/CLAUDE.md` against actual code in `src/backend/`. Check:
   - DynamoDB Schema Reference — are all tables, keys, GSIs, and fields current?
   - API Endpoint Catalog — are all controllers and endpoints listed? Any new/removed/changed?
   - **API response accuracy** — for each endpoint, verify documented status codes match actual behavior by tracing the full request path:
     1. Read the controller action to identify what the service layer can throw (e.g., `BusinessValidationException`, `EntityNotFoundException`, `ArgumentException`).
     2. Read `ExceptionHandlingMiddleware` to determine the HTTP status code each exception maps to.
     3. Read `CurrentUserMiddleware` for 401 scenarios (missing auth claims).
     4. Check `[Authorize]` / `[Authorize(Policy = "AdminOnly")]` attributes for 401/403 from the authorization pipeline.
     5. Document ALL possible status codes, not just the happy path. Phrases like "200 always" must be verified — if any code path in the service throws an exception that the middleware intercepts, document that status code.
   - Service patterns — do documented conventions match actual code?
   - Test patterns — do documented conventions match actual tests?
   - DI registration — any new services not mentioned?
   - AutoMapper profile — any new mappings not documented?

2. **Frontend agent:** Compare `src/frontend/CLAUDE.md` against actual code in `src/frontend/`. Check:
   - Component inventory — any new pages, components, services not reflected in the structure?
   - State management patterns — still using the documented approach?
   - Routing — any new routes or changed structure?
   - Syncfusion usage — any new components in use?
   - Test patterns — still matching documented conventions?

3. **Infrastructure agent:** Compare `src/infra/CLAUDE.md` against actual code in `src/infra/`. Check:
   - Stack structure — any new stacks or constructs?
   - Resource inventory — any new DynamoDB tables, Lambda functions, S3 buckets?
   - Pipeline changes — any new stages or modified flow?
   - Environment variable wiring — any new `AddEnvironment()` calls?

4. **E2E agent:** Compare `e2e/CLAUDE.md` against actual code in `e2e/`. Check:
   - Architecture section — does the file tree match actual directories and files?
   - Test spec inventory — any new test files not reflected?
   - Page objects — any new page object classes not listed?
   - Seed data — does the documented SEED structure match `seed-contracts.ts`?
   - Conventions — do documented patterns (fixtures, selectors, two-user model) match actual usage?
   - Troubleshooting — any new common issues not listed?

5. **Root agent:** Compare root `CLAUDE.md` against the full project. Check:
   - Domain Model section — are all entities listed with correct relationships?
   - Solution Structure — does the tree match actual directories?
   - Tech Stack — are versions current?
   - Development modes — still accurate?

## Phase 2: Drift Report
Compile agent findings into a drift report:

For each documentation file, list:
- **Accurate:** sections that match the code (no action needed).
- **Stale:** sections where code has changed but docs haven't (needs update).
- **Missing:** new code patterns/features with no documentation (needs addition).
- **Orphaned:** documentation for code that no longer exists (needs removal).

Present the drift report to the user before making changes.

## Phase 3: Update
After user reviews the drift report:
- Update stale sections to match current code.
- Add missing sections following the existing documentation style.
- Remove orphaned sections.
- Do NOT add speculative documentation — only document what exists in code.
- Do NOT change documentation structure unless necessary to accommodate new content.
- Keep documentation concise — reference code rather than duplicating it.

## Rules
- Never generate standalone README or documentation files unless explicitly requested.
- Update existing CLAUDE.md files in place.
- If a CLAUDE.md file doesn't exist for a directory that needs one, ask the user before creating it.
- Do not document implementation details that are obvious from reading the code — focus on conventions, patterns, and decisions that aren't self-evident.
- **API documentation must reflect the full request lifecycle, not just the controller action.** When documenting an endpoint's response codes, trace the request through the middleware pipeline (`ExceptionHandlingMiddleware`, `CurrentUserMiddleware`, authorization) and the service layer. If a service method can throw an exception that a middleware intercepts and converts to an HTTP error, that status code must be documented. Never write "always returns X" without verifying that no code path between the request and the response can produce a different status code.
