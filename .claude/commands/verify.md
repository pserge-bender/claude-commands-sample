Verify that a recently implemented feature is correct, secure, and documented.

Run this after `/implement` completes, or anytime you want to validate the current state of the codebase.

## Input
- If $ARGUMENTS points to a `.spec.md` file, read it as the feature spec. Also look for a matching plan file in the same directory (same name without `.spec` suffix) — read both. The spec provides requirements and use cases for traceability; the plan provides the implementation design.
- If $ARGUMENTS points to a plan `.md` file (non-spec), read it. Also look for a matching `.spec.md` — read both if found.
- If $ARGUMENTS is free text, treat it as a feature name/description. Search `docs/plans/` for a matching spec and plan.
- If $ARGUMENTS is empty, look at recent git changes (`git diff`, `git status`) to determine what was changed. Search `docs/plans/` for the most recently modified spec that matches the changed files. If no spec is found, skip requirements traceability checks (Phase 5 items that require the spec) and note this in the output.

## Phase 1: Build Verification
Build both backend and frontend, treating warnings as errors. Warning suppressions (e.g., `#pragma warning disable`, `// @ts-ignore`, `--warningsAsErrors false`) are NOT allowed unless the user has explicitly approved them for that specific case.

**Backend:**
```
dotnet build src/backend --warnaserror
```
- [ ] Backend builds with zero warnings and zero errors.
- [ ] No new `#pragma warning disable` or `[SuppressMessage]` directives added without explicit user approval.

**Frontend:**
```
cd src/frontend && npx ng build --configuration development
```
- [ ] Frontend builds with zero warnings and zero errors.
- [ ] No new `// @ts-ignore`, `// @ts-expect-error`, or `eslint-disable` directives added without explicit user approval.

If warnings exist: report each warning with file, line number, and message. Do NOT suppress them — fix the root cause. If a warning cannot be fixed (e.g., third-party type mismatch), report it and ask the user for explicit approval before suppressing.

## Phase 2: Dead Code & Unused Dependencies
Scan for dead code and unused dependencies in changed and related files:

**Backend:**
- [ ] No unused `using` directives in changed files.
- [ ] No unreferenced services, classes, or methods introduced by the changes.
- [ ] No NuGet packages added to `.csproj` files that are not imported/used in application code.

**Frontend:**
- [ ] No unused `import` statements in changed files.
- [ ] No npm packages in `package.json` `dependencies` that are not imported in any `src/` application code (test-only mocks do NOT count as usage — the package must be imported in non-test code).
- [ ] No exported functions, types, or components that are never referenced outside their own file.
- [ ] No orphaned test mocks for packages that are no longer dependencies (e.g., `vi.mock()` for an uninstalled package).

If dead code or unused dependencies are found: report each item. Do NOT remove them automatically — present the findings and let the user decide (some may be intentionally kept for future use).

## Phase 3: Test Suite
Run ALL tests across all affected layers:
```
dotnet test src/backend
cd src/frontend && npm test
```
**IMPORTANT:** Frontend tests MUST be run via `npm test` (which runs `ng test`), NEVER via `npx vitest run` directly. The `@angular/build:unit-test` builder bootstraps `TestBed.initTestEnvironment()` automatically — running Vitest directly skips this, causing all TestBed-based tests to fail with "Need to call TestBed.initTestEnvironment() first".

- [ ] All unit/integration tests pass — zero failures.
- [ ] No skipped or ignored tests that should be active.
- [ ] Test coverage exists for new logic (services, mappers, controllers).
- [ ] UI Tests covers new logic/pages.

### E2E Tests
E2E tests require a deployed environment. Prefer `--skip-deploy` if the e2e environment already has the latest code; use a full deploy only if tests fail due to stale code:
```
bash deploy-and-test.sh --profile rh --skip-deploy   # if env is up-to-date
bash deploy-and-test.sh --profile rh                  # full deploy + test
```
- [ ] All E2E tests pass — zero failures.
- [ ] New E2E test specs exist for new user-facing behavior.
- [ ] E2E tests cover both happy-path and cross-user data isolation (Alice vs Bob).
- [ ] `seed-contracts.ts` and `E2eSeeder.cs` are in sync.
- [ ] Page objects use `data-testid` selectors that exist in Angular templates.
- [ ] E2E tests follow `e2e/CLAUDE.md` conventions (fixtures, naming, two-user model).

If tests fail: report the failures with root cause analysis. Do NOT fix them — that is `/implement`'s job. If fixes are needed, include them in the verdict.

## Phase 4: Parallel Review
Launch the following review agents **in parallel** — they are independent and read-only:

1. **`pr-review-toolkit:code-reviewer`** — check all changed files against project conventions in CLAUDE.md files.
2. **`pr-review-toolkit:silent-failure-hunter`** — find swallowed errors, empty catch blocks, silent fallbacks in changed code.
3. **`pr-review-toolkit:pr-test-analyzer`** — evaluate test coverage quality and identify critical gaps.
4. **`pr-review-toolkit:type-design-analyzer`** — review any new types/DTOs/entities for proper encapsulation and invariants.
5. **`pr-review-toolkit:comment-analyzer`** — verify that new/changed comments are accurate and not stale.
6. **`coderabbit:code-reviewer`** — independent AI code review for additional coverage and a second perspective on code quality.

Collect all agent results before proceeding.

## Phase 5: Requirements Check
Compare the implementation against the plan/requirements:
- [ ] Every requirement is addressed — nothing was skipped.
- [ ] No extra scope — nothing was added beyond what was requested.
- [ ] Contracts are consistent between backend and frontend (endpoint paths, DTOs, status codes).
- [ ] Every Use Case from the spec's "Use Cases (E2E Scenarios)" section has a corresponding E2E test (if applicable).

## Phase 6: Security Review
Invoke the `security-guidance` skill for a structured security analysis of the changed code, then verify the following checklist:
- [ ] New infra resources follow least-privilege IAM.
- [ ] No resources are publicly accessible unless explicitly required.
- [ ] Backend endpoints require authentication (`[Authorize]` attribute).
- [ ] No sensitive data in logs, error messages, or client responses.
- [ ] Input validation at API boundary (controller level).
- [ ] No NoSQL injection vectors in DynamoDB queries.
- [ ] Frontend does not expose tokens or secrets in localStorage/URLs.
- [ ] CORS is not overly permissive.

## Phase 7: Documentation Review
Check whether documentation needs updating — **report** what is stale or missing, but do NOT update docs directly. Documentation changes are a separate concern and should be committed intentionally (via `/document` or manually), not bundled into an implementation commit.

Check the following for drift:
- [ ] `CLAUDE.md` (root) — stale if project-wide patterns or structure changed.
- [ ] `src/backend/CLAUDE.md` — stale if new services, controllers, DynamoDB tables, or conventions were added.
- [ ] `src/frontend/CLAUDE.md` — stale if new component patterns, services, or routing changes were made.
- [ ] `src/infra/CLAUDE.md` — stale if new constructs, resources, or deployment steps were added.
- [ ] `e2e/CLAUDE.md` — stale if new page objects, test specs, fixtures, or seed data conventions were added.
- [ ] API contracts — stale if new endpoints are undocumented.
- Do NOT check documentation unrelated to the changes.

## Output
Present a summary:
- **Build:** clean (zero warnings, zero errors) or list of warnings/errors per layer.
- **Dead code / unused deps:** clean or list of findings.
- **Unit/Integration Tests:** pass/fail count.
- **E2E Tests:** pass/fail count (or "not run — requires deploy" if environment not available).
- **Code review:** issues from each agent (grouped by severity).
- **Requirements:** covered / gaps found.
- **Security:** issues found or all clear.
- **Docs:** stale/missing sections found (if any) — suggest `/document` to fix.
- **Verdict:** ready to commit (suggest `/commit`), or list of items to fix first (suggest re-running `/implement` to address them).
