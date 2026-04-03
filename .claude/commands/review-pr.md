Perform a comprehensive code review of a pull request or the current branch's changes.

## Input
- If $ARGUMENTS is a PR number (e.g., `42`), fetch the PR details from GitHub.
- If $ARGUMENTS is a branch name, compare it against `master`.
- If $ARGUMENTS is empty, review the current branch's diff against `master`.

## Phase 1: Gather Context
1. Identify the changes:
   - For a PR: `gh pr diff {number}` and `gh pr view {number}`.
   - For a branch: `git diff master...HEAD` and `git log master..HEAD --oneline`.
2. Read the relevant `CLAUDE.md` files for each affected layer.
3. If a plan or spec exists in `docs/plans/` for this feature, read it — use it as the reference for what the code should do.

## Phase 2: Parallel Review
Launch the following review agents **in parallel**:

1. **`pr-review-toolkit:code-reviewer`** — check changed files against CLAUDE.md conventions, code quality, and logic errors.
2. **`pr-review-toolkit:silent-failure-hunter`** — find swallowed errors, empty catch blocks, silent fallbacks.
3. **`pr-review-toolkit:pr-test-analyzer`** — evaluate test coverage quality and identify critical gaps.
4. **`pr-review-toolkit:type-design-analyzer`** — review new types/DTOs/entities for encapsulation and invariants.
5. **`pr-review-toolkit:comment-analyzer`** — verify comments are accurate and not stale.
6. **`coderabbit:code-reviewer`** — independent AI code review for additional coverage.

## Phase 3: Project-Specific Checks
After agent results are collected, verify these RenoHouse-specific concerns:

### Cross-Layer Consistency
- [ ] REST contracts match between backend controllers and frontend services (endpoint paths, DTOs, status codes).
- [ ] New DynamoDB access patterns use keys/indexes efficiently — no table scans.
- [ ] AutoMapper mappings exist for any new DTOs.

### Auth & Security
- [ ] New endpoints have `[Authorize]` attribute.
- [ ] Feature works in both stub and OIDC auth modes.
- [ ] No secrets or tokens in client-side code, logs, or error messages.
- [ ] S3 operations use pre-signed URLs, not direct bucket access from the frontend.

### Frontend
- [ ] New pages/components follow existing Syncfusion + Angular patterns.
- [ ] `data-testid` attributes added for new interactive elements (needed by E2E tests).
- [ ] PostHog tracking added for new user-facing features.

### Infrastructure
- [ ] No destructive resource changes (table replacements, bucket deletions) without explicit justification.
- [ ] New Lambda environment variables wired via `api.Handler.AddEnvironment()`.

### Tests
- [ ] New backend logic has unit tests.
- [ ] New frontend logic has component/service tests.
- [ ] E2E test scenarios exist for new user-facing behavior.
- [ ] `seed-contracts.ts` and `E2eSeeder.cs` are in sync if seed data changed.

## Phase 4: Summary
Present a structured review:

- **Overview**: what the PR does (1-2 sentences).
- **Critical issues**: must fix before merge (bugs, security, data loss risks).
- **Suggestions**: improvements that are recommended but not blocking.
- **Nits**: style/formatting observations (low priority).
- **Test coverage**: adequate / gaps identified.
- **Verdict**: approve, request changes, or needs discussion.
