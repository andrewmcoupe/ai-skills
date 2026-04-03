---
name: feature-explainer
description: Generates a comprehensive README.md explainer for a feature, covering high-level purpose through to low-level implementation details. Works in two modes - from a prior design discussion (e.g. grill-me session) or by exploring an existing feature in the codebase. Use when the user says "write feature explainer", "explain this feature", "document this feature", or "feature explainer". Outputs to feature-explainers/{feature-name}/README.md.
---

# Feature Explainer

## Step 1 — Gather Info

Before doing anything, ask the user:

1. **Source** — "Are we documenting from our conversation above, or should I explore an existing feature in the codebase?"
2. **Feature name** — "What should I call this feature?" (suggest one inferred from context if possible)
3. **Directory naming** — "What naming convention for the directory — kebab-case (e.g. `car-configuration`), TitleCase (e.g. `CarConfiguration`), or something else?"
4. **Audience focus** — "Should this lean more towards non-technical readers, technical readers, or balanced for both?"

Wait for answers before proceeding.

## Step 2 — Gather Context

### From conversation
Collect all design decisions, requirements, trade-offs, and implementation details discussed in the conversation above.

### From existing codebase
Ask the user to point you to the relevant area of the codebase (directory, entry point, or key file). Then explore the feature thoroughly — read key files, trace data flow, understand component relationships, and identify business logic before drafting.

## Step 3 — Write the Explainer

Create a `README.md` inside `feature-explainers/{feature-name}/` in the current working directory. Use the naming convention the user chose.

### Document Structure

Flow from high-level to low-level. Adjust depth per section based on the audience focus chosen in Step 1.

#### 1. Overview
- What the feature is in plain language
- What problem it solves and why it exists
- Who it's for (end users, admins, developers, etc.)

#### 2. How It Works
- User-facing behaviour and key workflows
- Important business rules and edge cases
- What happens when things go wrong (error states, fallbacks)

#### 3. Architecture
- Key components and how they connect
- Data flow — what moves where and why
- State management approach if applicable
- External dependencies or integrations

#### 4. Implementation Details
- Key files and their responsibilities (use relative paths from project root)
- Important functions, hooks, or modules and what they do
- Data models / schemas if relevant
- API endpoints or server actions if relevant

#### 5. Design Decisions
- Decisions made and **why** — not just what was chosen, but what was rejected and the reasoning
- Trade-offs accepted
- Known limitations or future considerations

#### 6. Quick Reference
- Glossary of domain terms, if any jargon exists
- Key configuration or environment variables, if applicable

## Rules

1. **Derive from source** — every detail must trace back to the conversation or the codebase. Do not invent details.
2. **Plain language first** — lead every section with the simplest explanation before adding technical depth. A product manager should understand sections 1–2 without help.
3. **Be specific** — prefer "the car configurator validates that selected engine options are compatible with the chosen chassis" over "it validates inputs".
4. **No code blocks for explanation** — describe behaviour in prose. Only use code snippets if a specific pattern or API shape was explicitly agreed upon or is essential for clarity.
5. **Present before writing** — show the user the proposed content and get confirmation before writing to the file.
6. **Updating an existing explainer** — if a README.md already exists at the path, read it first. Merge new information rather than overwriting. Flag any conflicts with the user.
7. **Keep it standalone** — the document should make sense without needing to read the source code. Someone onboarding should be able to read this and understand the feature fully.
8. **Link to PRD if exists** — if a `feature-plans/{feature-name}/PRD.json` exists, add a note at the top linking to it for acceptance criteria.
9. **Omit empty sections** — if a section has no relevant content (e.g. no domain jargon for Quick Reference), leave it out rather than writing a placeholder.
