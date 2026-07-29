---
name: prd-to-task-graph
description: Turn a feature idea, brief, or PRD into an executable dependency graph of tasks (graph engineering) and complete it phase by phase with approval gates. Use this whenever the user wants to build a feature or project from a spec, asks to "apply graph engineering", mentions task graphs, DAGs, PRDs, decomposing work for an agent, or says things like "let's build X" for anything with more than a handful of steps — even if they don't name this skill.
---

# PRD to task graph

Complete a build request through four phases: interrogate → PRD → task graph → execute. Stop for the user's explicit approval between every phase. The gates exist because each phase is cheap to review and expensive to get wrong downstream: a wrong assumption caught at the PRD stage costs thirty seconds; caught during execution it costs the whole run.

Enter at whatever phase the input allows. If the user hands you a finished PRD, start at Phase 3. If they hand you a rough idea, start at Phase 1. Never skip a phase's _output_ — every run must produce a PRD and a `task_graph.json`, even if abbreviated — but don't re-interrogate things the input already answers.

## Phase 1 — Interrogate

**Delegate if possible.** If the `grill-with-docs` skill (mattpocock/skills) is installed, run the interrogation through it instead of the fallback below. Its outputs become required inputs downstream: the CONTEXT.md glossary and any ADRs it produces feed Phase 2 (write the PRD in the glossary's terms — one name per concept, no synonyms), and each Phase 3 task should list the ADR files relevant to it so Phase 4 workers receive them as per-task context. A worker holding the ADR that says "we chose X over Y because Z" cannot accidentally re-litigate a settled decision mid-task — that is drift prevention for free.

**Fallback interrogation** (no grill skill installed): interview the user until every feature has two things pinned down:

1. **A verification story** — "how would we check this is done?" This becomes the acceptance criterion. Push past vague answers: "it should work" is not verifiable; "the endpoint returns paginated results and the test suite passes" is.
2. **An ordering story** — "does this need anything else to exist first?" This becomes a dependency edge.

Also establish: target stack and constraints, what is explicitly out of scope, and what "done" means for the project as a whole (the final integration gate).

Interview mechanics:

- **One question at a time**, in dependency order — resolve the decisions that other decisions hang on first. Never dump a questionnaire.
- **Offer a recommended answer with every question**, so the user can reply "yes" or adjust rather than compose from scratch. Recommendations make the interview fast without making it shallow.
- **Inspect, don't ask**: if the answer already exists in the codebase or the provided materials, go read it. Never ask the user to explain what already exists.
- Prefer questions that expose contradictions or gaps over questions that merely collect preferences.
- Record hard-to-reverse decisions as they crystallise — a one-paragraph ADR-style note (decision, alternatives, why) per one-way door. These notes travel to Phase 4 workers the same way full ADRs do.

When the answers stabilize, summarize the whole picture back in a few sentences and ask "is this right?" — that confirmation is the gate to Phase 2.

## Phase 2 — Write the PRD

Produce a PRD from the interrogation. Keep it as short as the project allows — the PRD is a contract, not a novel. If Phase 1 produced a glossary, use its terms exactly; if it produced ADRs or decision notes, reference them rather than re-explaining the reasoning. Use this structure:

```
# [Project name]
## Goal (2-3 sentences)
## Out of scope
## Features
For each feature:
- What it does
- Acceptance criteria (concrete, testable)
- Depends on (other features, if any)
## Technical constraints (stack, style, integrations)
## Definition of done (the final integration gate)
```

Every feature must carry at least one testable acceptance criterion. If you cannot write one, the interrogation was incomplete — go back and ask, do not invent one. Present the PRD and wait for approval before Phase 3.

## Phase 3 — Build the task graph

Decompose the PRD into `task_graph.json`:

```json
{
  "version": 1,
  "source": "PRD.md",
  "graph_validation": {
    "task_count": 2,
    "acyclic": true,
    "missing_dependencies": [],
    "topological_batches": [["data-model"], ["auth-flow"]]
  },
  "prd_section_coverage": {
    "User authentication": "auth-flow",
    "Core data model": "data-model"
  },
  "ambiguities": [
    {
      "id": "session-duration",
      "description": "PRD does not specify token lifetime or refresh policy",
      "blocks_tasks": ["auth-flow"],
      "state": "blocking",
      "resolution": null
    }
  ],
  "pending_approvals": [],
  "tasks": [
    {
      "id": "auth-flow",
      "title": "Authentication flow",
      "description": "What to build, scoped to this task only",
      "dependencies": ["data-model"],
      "acceptance_criteria": ["JWT issued on valid login", "auth tests pass"],
      "adrs": ["docs/adr/0003-jwt-over-sessions.md"],
      "outputs": "",
      "status": "pending"
    }
  ]
}
```

The extra top-level blocks are cheap proofs the user can check at a glance: `graph_validation` shows the parallelism (or its absence) explicitly, `prd_section_coverage` proves every PRD section landed in exactly one task, and `ambiguities` records what the graph refuses to guess. Populate all three at graph time.

### Ambiguity lifecycle

Ambiguities are first-class records, not comments — each has a `state` that must stay truthful for the whole run:

- **blocking**: unresolved; the listed tasks cannot be marked `done` while it stands.
- **resolved**: the user decided. Write who decided what into `resolution` (e.g. "user approved 30-day token lifetime, 2026-07-28") before any blocked task completes.
- **deferred**: the task built a versioned or config-gated mechanism around the unknown — the code is real, the values are still owed. Deferring is a legitimate engineering move (build the scorer, parameterise the weights; build the shutdown, gate the timestamps), but it must be recorded: set `resolution` to what was built and what is still owed, and add an entry to `pending_approvals` naming exactly what the user must supply or sign off.

A task blocked by a `deferred` ambiguity finishes as `done-gated`, not `done` (see Phase 4). The rule that keeps the file honest: whenever an ambiguity is resolved or worked around, update its entry _before_ updating the affected task's status. A file where an ambiguity says "blocking" and its task says "done" is lying to whoever reads it next.

Watch especially for acceptance criteria containing the word "approved" — approval means the _user_ decided, and an agent can quietly satisfy "approved scoring contract" by writing the contract itself. Any criterion needing user sign-off that hasn't received it routes through `deferred` + `pending_approvals`, never through silent self-approval.

### Decomposition rules

- A task is well-scoped when it has one clear deliverable, is independently verifiable, and produces outputs another task could consume. If you can't state its acceptance criteria without referencing another task's internals, the boundary is wrong.
- Sizing: a task should be one coherent working session, not one line of code and not "build the backend". Too granular multiplies coordination overhead per node; too coarse collapses back into a single loop.
- Every leaf-level feature in the PRD maps to exactly one task. Anything ambiguous gets flagged to the user, not guessed.
- The graph must be a DAG. Verify there are no cycles before presenting it.

### Dependency discipline

Add an edge only if the task would _literally fail_ without the other task's output — not "it would be nice to do X first". False edges silently serialize the graph and destroy the parallelism that justifies it. Check for these anti-patterns before presenting:

- **Everything-depends-on-setup**: a setup node that every task points to. Usually only tasks that genuinely consume the scaffolding need the edge.
- **False sequential chains**: A→B→C where B and C only share a theme, not data. Break the chain.
- **The hidden monolith**: one node whose description contains "and" three times. Split it.
- **Orphaned cross-cutting concerns**: never make "error handling", "logging", or "tests" their own tasks — fold them into each feature task's acceptance criteria.
- **Fan-in on implementations when interfaces would do**: a node (notification centre, analytics, reporting) that waits on seven upstream tasks usually only needs their _contracts_ — event types, DTO shapes — not their finished code. Have early tasks emit typed contracts as part of their outputs, and let the consumer build against the contracts in parallel. Wide fan-in nodes are where parallelism goes to die; check each one for edges that could be loosened to a contract dependency.

Present the graph as an indented tree that makes the parallel groups visible, and wait for approval before Phase 4. Invite the user to challenge the edges specifically — edge review matters more than task review.

## Phase 4 — Execute

1. Walk the graph in topological order. Whenever multiple tasks have all dependencies satisfied, treat them as a parallel batch (run them as parallel subagents/runs if the environment supports it; otherwise back-to-back with fresh context each).
2. **Context discipline** — this is the point of the whole structure: when starting a task, work only from (a) that task's entry in `task_graph.json`, (b) the `outputs` summaries of its direct dependencies, and (c) the ADRs or decision notes listed in the task's `adrs` field. Do not re-read the full PRD or unrelated code unless the task's description requires it. This is what prevents the accumulating-context replay tax of a long loop.
3. **Gate every task**: a task is done only when all its acceptance criteria pass, including running the relevant tests. On failure, retry within the task up to 2 times. Still failing → set status "failed", skip its downstream tasks, continue unaffected branches, and report the failure at the end. Never mark a task done on the strength of your own claim that the code looks right — the criteria must actually be checked.
4. **Statuses**: `pending` → `in-progress` → one of `done`, `done-gated`, or `failed`. Use `done-gated` when the code-level criteria pass but the task depends on a `deferred` ambiguity — mechanism built, real values or user sign-off still owed. A `done-gated` task unblocks downstream work like `done` does, but the final integration gate cannot pass while any task remains `done-gated`: production activation waits for the pending approvals to clear.
5. After each task, update `task_graph.json`: status, a 2-3 sentence `outputs` summary describing what was built and the interfaces exposed (so downstream tasks can consume it without re-reading code), and any ambiguity-state or `pending_approvals` changes — ambiguity entries update _before_ task statuses do. The file doubles as resumable state and as a live dashboard the user can audit mid-run: if a run is interrupted, resume by re-reading it and continuing from the current statuses.
6. **Batch-boundary check-ins**: at the end of each topological batch (or every 3-4 tasks in near-sequential graphs), give the user a short status digest — tasks completed, anything `done-gated`, and the current `pending_approvals` queue. Surface the queue every time it is non-empty; the user should never have to discover what they owe by reading sixteen output summaries. These check-ins are informational, not approval gates — keep executing unless the user intervenes or an escalation rule triggers.
7. When all branches finish, run the full test suite as the integration gate from the PRD's definition of done, then report: tasks completed, tasks failed and why, deviations from the PRD, and the final `pending_approvals` queue with what each item is blocking.

## Verification hierarchy

The orchestrator owns the graph; workers own their tasks. That split only works if checking a task's completion never reduces to believing the worker's report — workers are systematically optimistic about their own output, and a self-graded gate is no gate. Three rules keep verification real:

1. **The orchestrator is the sole writer of `task_graph.json`.** Workers propose their `outputs` summary and claim completion; the orchestrator verifies first, then commits the status and summary. A worker never marks itself done.
2. **Every acceptance criterion carries at least one runnable check** — a command with an exit code (`pnpm test`, a curl assertion, a schema validation script). Write criteria this way at graph time: verification then transfers from LLM judgment to the machine. To gate a task, the orchestrator re-runs the acceptance commands itself in a clean context and reads the exit codes; it does not ask the worker whether they passed.
3. **The reviewer is never the worker.** For qualities commands can't check (design sanity, PRD intent), review happens with fresh eyes: the orchestrator judging with only the task spec and the diff in context, a dedicated reviewer node in the graph, or — strongest — a different model reviewing the diff. If the whole run is one model in one session, treat LLM review as weak and push everything possible into rule 2; cross-model review is the cheapest way to buy genuine independence.

Anything no command and no reviewer can verify — "is this what the user actually wanted" — belongs in `pending_approvals`, not in a status field.

## Failure and escalation

If the same task fails twice for the same reason, or two tasks' acceptance criteria turn out to contradict each other, stop executing and bring it to the user — that is a spec problem, not a code problem, and it belongs back in Phase 1 or 2. Update the PRD and the graph before resuming so the artifacts stay truthful.
