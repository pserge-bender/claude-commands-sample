Collect context and produce a clear, complete `.spec.md` file that `/design` can consume directly.

## Input
- If $ARGUMENTS points to a file, read it as initial context (partial spec, notes, investigation, etc.).
- If $ARGUMENTS is free text, treat it as an initial feature description.
- If $ARGUMENTS is empty, ask the user what feature they want to specify.
- The user may combine approaches: provide a file AND free-text clarifications.

## Phase 1: Gather Context
1. Read the initial input carefully.
2. Read the relevant `CLAUDE.md` files to understand the project:
   - `CLAUDE.md` (root) — domain model, tech stack, project structure.
   - `src/backend/CLAUDE.md` — backend architecture, DynamoDB patterns.
   - `src/frontend/CLAUDE.md` — frontend conventions, components.
   - `src/infra/CLAUDE.md` — infrastructure, deployment.
   - `e2e/CLAUDE.md` — existing E2E test coverage, patterns, conventions.
3. Check `docs/plans/` for existing specs (`*.spec.md`) and plans that relate to the feature — avoid duplicating or contradicting prior decisions.
4. If the input references other files (investigations, existing code, docs), read them.

## Phase 2: Analyze & Identify Gaps
From the initial context, extract:
- **Goal**: what the feature should achieve for the user.
- **Scope**: which layers are likely affected (infra, backend, frontend, e2e).
- **Constraints**: any explicit constraints or non-goals mentioned.
- **Unknowns**: requirements that are ambiguous, missing, or could be interpreted multiple ways.

For each unknown, determine whether you can resolve it by:
1. **Reading the codebase** — if the answer is in existing code/config, look it up instead of asking.
2. **Making a reasonable assumption** — based on project conventions and domain knowledge.
3. **Asking the user** — only for genuine business decisions that cannot be inferred.

## Phase 3: Interactive Clarification
Present your understanding to the user in this structure:

### What I understand
Summarize the feature goal, scope, and any constraints — in your own words, not just repeating the input.

### Assumptions I'm making
List assumptions you've made based on project conventions or domain reasoning. For each, explain **why** you assumed it. Ask the user to confirm, correct, or expand.

### Questions
Ask only questions that you genuinely cannot answer from the codebase or reasonable inference. Group them by topic. Limit to the most important questions — do not overwhelm the user. Prioritize questions whose answers would change the spec significantly.

**Wait for the user to respond before proceeding.** Iterate this phase if the answers reveal new unknowns. Keep rounds to a minimum — aim for 1-2 rounds max.

## Phase 4: Draft the Spec
Once you have enough clarity, produce the `.spec.md` file. Follow this structure:

```markdown
# {Feature Name}

## Goal
One-paragraph description of what this feature achieves and why it matters.

## Scope
Which layers are affected: infrastructure, backend, frontend, e2e. Briefly state what each layer needs to do.

## Requirements
Numbered list of concrete, testable requirements. Each requirement should be specific enough that `/design` can produce an implementation plan without further questions.

## Use Cases (E2E Scenarios)
List user-facing use cases that exercise the feature end-to-end. Each should be a concrete scenario that can be converted into a Playwright E2E test:
- Format: "As [user], when I [action], then [expected outcome]"
- Cover the happy path, key validation/error paths, and data-isolation between users (if the feature involves user-scoped data).
- Reference existing e2e patterns from `e2e/CLAUDE.md` where applicable.

## Constraints
- Explicit constraints, non-goals, or things intentionally left out.
- Technical constraints (e.g., must use existing table, must not break X).

## Assumptions
- Key assumptions made during clarification, with brief rationale.
- These help `/design` understand the reasoning behind requirements.

## Open Questions (if any)
- Questions that were deliberately deferred or need input from others.
- Mark these clearly so `/design` knows to flag them.
```

Keep the spec concise and actionable. Every requirement should inform a design decision. Avoid vague language like "should be easy to use" — translate user intent into concrete behavior. Ensure the Use Cases section contains enough detail for `/design` to produce concrete E2E test scenarios.

## Phase 5: Review & Save
1. Present the draft spec to the user.
2. Ask: "Does this spec capture everything? Any changes before I save it?"
3. After explicit approval, save to `docs/plans/{feature-name}.spec.md`.
   - If the user specified a different output path, use that instead.
   - Use kebab-case for the filename, derived from the feature name.
4. Tell the user they can now run `/design docs/plans/{feature-name}.spec.md` to produce the implementation plan.
