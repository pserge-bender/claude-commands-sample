Systematically debug a bug, test failure, or unexpected behavior in the RenoHouse codebase.

## Input
- If $ARGUMENTS describes a bug or error, treat it as the problem statement.
- If $ARGUMENTS points to a file, read it for context (error logs, screenshot, stack trace).
- If $ARGUMENTS is empty, check `git diff` and recent test output for failures.

## Phase 1: Understand the Problem
1. Read the problem statement / error message carefully.
2. Classify which layer is likely affected:
   - **Frontend** (Angular): UI rendering, routing, component errors, HTTP client issues, auth flow.
   - **Backend** (.NET): controller errors, service exceptions, DynamoDB access, middleware, serialization.
   - **Infrastructure** (CDK): deployment failures, resource configuration, IAM permissions.
   - **E2E** (Playwright): test failures, selector issues, timing, seed data mismatches.
   - **Cross-layer**: contract mismatches (frontend ↔ backend DTOs), auth mode issues (stub vs OIDC), proxy/CORS.
3. Read the relevant `CLAUDE.md` files for the affected layer(s).

## Phase 2: Reproduce & Gather Evidence
Collect evidence **before forming hypotheses**. Launch parallel investigations as appropriate:

### Frontend issues
- Check browser console errors via `chrome-devtools-mcp:chrome-devtools` skill if the app is running.
- Read the component/service code at the error location.
- Check `proxy.conf.json` if it's an API connectivity issue.
- Verify the correct environment file is being used (`environment.ts` vs `environment.prod.ts` vs `environment.local-oidc.ts`).

### Backend issues
- Read the stack trace and identify the failing line.
- Check DynamoDB table schema and access patterns against `src/backend/CLAUDE.md`.
- Verify `LocalDevMiddleware` behavior if the issue is auth-related in local dev.
- Check `launchSettings.json` for environment variable configuration.

### Infrastructure issues
- Run `cdk diff` to see pending changes.
- Check CloudWatch logs via `awslabs.cloudwatch-mcp-server` for Lambda runtime errors.
- Check CloudTrail via `awslabs.cloudtrail-mcp-server` for IAM permission denials.

### E2E test failures
- Read the test spec and the page object it uses.
- Verify `seed-contracts.ts` matches `E2eSeeder.cs`.
- Check if `data-testid` selectors in the test exist in Angular templates.
- Review Playwright trace/screenshot if available.

### General
- `git log --oneline -10` — check recent changes that could have introduced the bug.
- `git diff` — check uncommitted changes.
- Use `microsoft-docs:microsoft-code-reference`, `syncfusion-angular-assistant` MCP, or `angular-cli` MCP to verify API usage if the bug involves a library call.

## Phase 3: Hypothesize & Test
1. Form **at most 3 hypotheses** ranked by likelihood, based on the evidence.
2. For each hypothesis:
   - State what you expect to find if the hypothesis is correct.
   - Run a targeted check (read specific code, run a specific test, query a log).
   - Confirm or eliminate the hypothesis.
3. Do NOT jump to fixes. Confirm the root cause first.

## Phase 4: Fix
1. Make the **minimal change** that addresses the root cause.
2. Run the relevant tests:
   - Backend: `dotnet test src/backend`
   - Frontend: `cd src/frontend && npm test`
   - E2E: note that E2E requires deployment — flag this to the user.
3. Verify no regressions — previously passing tests must still pass.

## Phase 5: Report
Present:
- **Root cause**: what was wrong and why.
- **Fix**: what was changed (with file:line references).
- **Verification**: which tests passed.
- **Prevention**: if applicable, suggest a test or check that would catch this earlier.

## RenoHouse-Specific Debugging Reference

| Symptom | Likely cause | Where to look |
|---------|-------------|---------------|
| 401 on API calls locally | Backend not running or wrong port | `localhost:5008`, `proxy.conf.json` |
| "User not found" in stub mode | `LocalDevMiddleware` not injecting claims | `ASPNETCORE_ENVIRONMENT` must be `Development` |
| DynamoDB "table not found" | Wrong table name or missing env var | `MainStack.cs` `AddEnvironment()`, `appsettings.json` |
| Blank page after login | OIDC callback misconfigured | `OidcAuthService`, Cognito app client settings |
| E2E test finds wrong data | Seed data mismatch | `E2eSeeder.cs` vs `seed-contracts.ts` |
| Angular build fails | TypeScript version or Syncfusion import | `package.json`, `tsconfig.json` |
| PostHog events missing | Wrong API key or environment config | `environment.*.ts`, `PosthogService` |
