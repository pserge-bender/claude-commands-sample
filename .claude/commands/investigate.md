Investigate a technical question or problem and produce a structured comparison of alternatives.

## Input
- If $ARGUMENTS points to a `.spec.md` or `.md` file, read it as the investigation brief.
- Otherwise treat $ARGUMENTS as a brief investigation question.
- If neither is provided, ask the user what to investigate.

**Investigation brief format** (for file-based input, save to `docs/investigations/{topic}.spec.md`):

```markdown
# Investigation: {title}

## Question
What specific question or problem needs answering?

## Context
Why is this relevant now? What triggered the investigation?

## Constraints
Any hard requirements, limits, or non-negotiables.

## Scope
What is in/out of scope for this investigation.
```

## Phase 1: Understand
1. Read the investigation brief/question carefully.
2. Read the relevant `CLAUDE.md` files to understand current architecture and conventions.
3. **Use parallel `Explore` agents** to gather facts from the codebase relevant to the question — existing patterns, dependencies, versions, usage sites, etc.
4. Codebase is the source of truth — if CLAUDE.md and code disagree, trust the code.

## Phase 2: Clarify
- If the question is ambiguous, the scope is unclear, or critical context is missing — ask the user targeted questions before proceeding.
- Specifically clarify:
  - What outcome they're optimizing for (speed, cost, maintainability, correctness, etc.)
  - Any options they've already considered or ruled out
  - Timeline or urgency constraints
- Do NOT guess business priorities — ask.

## Phase 3: Investigate
1. Research each viable alternative thoroughly:
   - Read relevant code, configs, and dependencies in the codebase.
   - Use `microsoft-docs:microsoft-docs` skill for .NET / Azure / ASP.NET Core-specific research.
   - Use the `syncfusion-angular-assistant` MCP and `angular-cli` MCP for Angular/Syncfusion-specific lookups.
   - Use `awslabs.aws-documentation-mcp-server` for AWS service documentation.
   - Use `WebSearch` or `WebFetch` for broader external information (comparisons, known issues, community discussions) when the codebase and dedicated doc tools aren't sufficient.
   - Check compatibility with the current tech stack (Angular 21, .NET 10, AWS CDK, DynamoDB).
2. For each alternative, gather concrete evidence: version compatibility, migration effort, performance characteristics, community support, licensing.
3. Identify at least **two** viable alternatives. If only one option exists, explain why and still present it against the status quo / "do nothing" baseline.

## Phase 4: Output

Present results to the user, then after approval save to `docs/investigations/{topic}.md` using this structure:

```markdown
# Investigation: {title}

## Question
{the question being investigated}

## Context
{why this matters, current state, what triggered the investigation}

## Alternatives

### Alternative 1: {name}

**Description:** {what this option is and how it works}

**Pros:**
- {concrete benefit with evidence}
- ...

**Cons:**
- {concrete drawback with evidence}
- ...

**Effort:** {estimated scope: small/medium/large + what's involved}

### Alternative 2: {name}

**Description:** ...

**Pros:** ...

**Cons:** ...

**Effort:** ...

<!-- repeat for additional alternatives -->

## Comparison Matrix

| Criterion | Alt 1 | Alt 2 | ... |
|-----------|-------|-------|-----|
| {criterion relevant to the question} | ... | ... | ... |

## Recommendation
{which alternative and why, given the stated constraints and priorities}
```

## Rules
- Every claim in pros/cons must be grounded in evidence (codebase facts, documentation, benchmarks) — no vague assertions.
- Effort estimates should reference specific files/areas that would need changes.
- The comparison matrix criteria should reflect what the user said they care about in Phase 2.
- If the investigation reveals the question itself is wrong (e.g., based on a false assumption), say so directly before presenting alternatives.
- Do NOT make the decision for the user — present the trade-offs clearly and let them choose.
- Keep the output actionable: after reading it, the user should be able to pick an alternative and hand it to `/design` for implementation planning.
