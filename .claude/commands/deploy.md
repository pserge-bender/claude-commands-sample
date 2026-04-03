Deploy RenoHouse to an AWS environment.

## Input
- $ARGUMENTS should specify the target environment: `dev`, `dev-am`, or `e2e`.
- If $ARGUMENTS includes `--test` or is `e2e`, run E2E tests after deployment.
- If $ARGUMENTS includes `--skip-deploy`, run E2E tests only (no deployment).
- If $ARGUMENTS is empty, ask the user which environment to deploy to.

## Rules
- **Never deploy to `uat` or `prod`** — these are CI/CD-only (CodePipeline). If the user asks, remind them to use AWS Console → CodePipeline → "Release change".
- Always use `--profile rh`.
- Run a build check before deploying to catch errors early.

## Pre-Deploy Checks
1. Check for uncommitted changes: `git status`. Warn the user if there are unstaged changes — they won't be included in the deploy.
2. Build both layers to catch compile errors:
   ```
   dotnet build src/backend
   cd src/frontend && ng build
   ```
3. If either build fails, stop and report the error. Do not deploy broken code.

## Deploy
Based on the target environment:

### `dev` or `dev-am`
```bash
cd src && bash deploy.sh {env} --profile rh
```
Report the output. Highlight any CDK diff warnings about resource replacements or deletions.

### `e2e`
```bash
bash deploy-and-test.sh --profile rh
```
This deploys the `e2e` environment AND runs Playwright E2E tests. Report both deployment and test results.

### `e2e` with `--skip-deploy`
```bash
bash deploy-and-test.sh --profile rh --skip-deploy
```
Runs E2E tests only against the already-deployed `e2e` environment.

## Post-Deploy
- Report deployment status (success/failure).
- If E2E tests ran, report pass/fail counts and any failures with details.
- If deployment failed, check CloudWatch logs via `awslabs.cloudwatch-mcp-server` for Lambda errors, or CloudTrail via `awslabs.cloudtrail-mcp-server` for permission issues.
