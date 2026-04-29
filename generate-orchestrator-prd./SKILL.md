---
name: generate-orchestrator-prd
description: Interview the user about a feature or project, resolve all design decisions, then generate a PRD.md optimised for the orchestrator's ingest system. Use when the user says "generate prd", "create prd", "write prd", or "plan a feature".
---

You are helping the user create a PRD (Product Requirements Document) that will be ingested by the orchestrator. The orchestrator extracts propositions, groups them into tasks, and assigns them to AI coding agents — so the PRD must be clear, specific, and actionable.

## Phase 1: Grill the user

Interview the user relentlessly about every aspect of their plan until you reach shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one by one.

Rules:
- Ask questions **one at a time**
- For each question, provide your recommended answer based on what you've learned so far
- If a question can be answered by exploring the codebase, explore the codebase instead of asking
- Cover: scope, constraints, edge cases, error handling, data model, UI/UX, dependencies, and anything ambiguous
- Do not move on from a topic until the decision is resolved
- Push back on vague answers — ask for specifics

When all branches are resolved, say: **"I have enough to write the PRD. Ready?"**

Wait for confirmation before proceeding.

## Phase 2: Generate the PRD

Write a `PRD.md` file in the current working directory with the following structure:

```markdown
# [Feature title — clear, concise]

## Overview

[2-3 sentence summary of what this feature does and why it exists.]

## [Task section title — action-oriented, e.g. "Add rate limiter middleware"]

[Detailed requirements for this task. Each bullet should be a single, testable behaviour:]

- Specific requirement one
- Specific requirement two
- Edge case handling
- Error scenarios

## [Next task section]

- ...

## Implementation Touchpoints

| File | Change |
|---|---|
| `path/to/file.ts` | Description of what changes in this file |
| `path/to/other.ts` | Description of what changes in this file |

## Out of Scope

- Things explicitly excluded from this PRD
- Prevents scope creep during implementation
```

### PRD rules

1. **Each `##` section becomes a separate task** assigned to an AI agent working in its own git worktree. Write each section as if it will be read by someone with no context beyond the section itself and the overview.

2. **Be specific and testable.** Every bullet should describe a single behaviour that can be independently verified. "Handle errors gracefully" is bad. "Return a 429 response with a Retry-After header when the rate limit is exceeded" is good.

3. **Order sections by dependency.** If task B depends on task A, section A should come before section B. The orchestrator will detect dependencies from the content.

4. **Do not create standalone test tasks.** The orchestrator has a separate test-author phase. Each section should describe the implementation — tests are handled automatically.

5. **Include enough context per section.** Reference file paths, function names, data shapes, and API contracts. The agent implementing this section only sees this section and the codebase — not the other sections.

6. **Include an Implementation Touchpoints table.** List every file that will be created or modified, with a brief description of the change. This gives agents a map of the codebase surface area and helps the orchestrator detect conflicts between tasks.

7. **Include an Out of Scope section.** Explicitly state what this PRD does not cover. This prevents agents from over-building and keeps tasks focused.

8. **Flag unknowns.** If something was unresolved during the interview, note it explicitly so the orchestrator can raise a pushback during ingest.

9. **Keep sections focused.** Each section should be completable in a single focused session. If a section feels too large, split it. If it feels too small, merge it with a related section.

After writing the file, tell the user: **"PRD written to PRD.md. Run the orchestrator and ingest it to create tasks."**
