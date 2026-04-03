Create a git commit for the current changes.

## Rules:
- Run build checks before committing based on which files changed:
  - Backend files changed → `dotnet build src/backend`
  - Frontend files changed → `cd src/frontend && ng build`
  - E2E files changed → `cd e2e && npx tsc --noEmit`
  - If any build fails, fix the issue or stop.
- Stage only relevant files. Never stage .env, .env.e2e, credentials, or cdk.context.json.
- Write a conventional commit message: type(scope): description
  - Types: feat, fix, refactor, test, docs, chore, style
  - Scopes: frontend, backend, infra, pipeline, e2e
  - Example: feat(frontend): add budget alert notifications
  - Example: test(e2e): add project wizard E2E tests
  - For cross-scope changes (e.g., frontend `data-testid` + e2e tests), use the primary scope or split into separate commits.
- Keep the subject line under 72 characters.
- Add a body only if the "why" isn't obvious from the subject.
- If $ARGUMENTS contains `--push`, push to the remote after committing. Otherwise do NOT push.
- Any remaining $ARGUMENTS (after removing `--push`) should be used as context for the commit message.
