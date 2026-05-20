# Parallel Execution Rules

These rules govern when and how to run parallel agents in the product development pipeline. Read this before implementing any step marked [PARALLEL] in `build-product.md`.

---

## When to Parallelize

Parallelize when ALL of the following are true:
1. The steps read from the same inputs (no step depends on the other's output)
2. Each step writes to a different output file
3. The steps are functionally independent (different reviewers, different validators)

Do NOT parallelize when:
- Step B needs Step A's output to run correctly
- Steps write to the same file (race condition)
- The user needs to review Step A before Step B begins
- Steps require interactive user input mid-execution

---

## The Three Parallel Blocks

### Block 1 — Dual Review (Steps 3+4)

**Why parallel:** Product Review and CTO Review both read the PRD and write to separate files. Zero dependency.

**How to execute:**
```
Spawn two agents simultaneously:
- Agent A: Product Review → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/product-review/[feature-name]-product-review.md
- Agent B: CTO Review    → ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[feature-name]-technical-review.md

Each agent receives: path to the PRD, its review instructions, its output path.
Each agent operates independently with no shared state.
```

**After both complete — synthesize before applying fixes:**
- List where both reviews agree → apply immediately, high confidence
- List where reviews conflict → surface to user at Gate 1 as a judgment call
- List items only one review raised → apply with a note

### Block 2 — Three-Way Validation (Step 8)

**Why parallel:** AC compliance, design match, and NFR checks each examine different dimensions of the same build. No ordering dependency.

**How to execute:**
```
Spawn three agents simultaneously:
- Agent A: AC Compliance  — check every acceptance criterion in the PRD against the build
- Agent B: Design Match   — check current UI against the design catalog (if exists)
- Agent C: NFR Check      — check performance thresholds, error/loading/empty states,
                            mobile layout, auth enforcement, dark mode (if applicable)

Each agent receives: PRD path, phase number, design catalog path (Agent B), output path.
```

**After all three complete — merge into one report:**
- Overall result: Pass / Pass with notes / Fail
- If any Fail items exist: fix before proceeding to Gate 3

### Block 3 — Parallel Export at Step 11 (Work pipeline)

**Why parallel:** Jira (always), Google Drive sync (optional), and Confluence publishing (optional) all read the same inputs (the approved breakdown + the feature folder) and write to separate external systems. Zero dependency between them.

**Pre-flight (before spawning):** Ask the PM at the Step 11 pre-flight which optional exporters to enable, and collect required inputs (Drive folder, Confluence space) before the parallel block runs.

**How to execute:**

```
Determine which exporters the PM enabled.
Spawn one agent per enabled target simultaneously:
- 11a — Jira Export        (always)
- 11b — Drive Sync         (if enabled)
- 11c — Confluence Publish (if enabled)

Each agent receives its own inputs and writes to its own external system.
Each agent operates independently. If one fails, the others continue —
no global rollback.
```

**After all enabled agents complete — synthesize:**
- Report all created URLs (Jira Epic, Drive folder, Confluence page) in one block.
- Report any per-agent failures with their original error messages.
- Update `_pipeline-state.md` with whichever URLs succeeded.

---

## Phase Pipelining (Advanced)

After Phase N execute completes and before Gate 3:
- The main agent continues with: validate → update PRD → present Gate 3
- A background agent begins: read Phase N+1 scope from PRD, produce screen inventory draft

**Result:** When the user approves Gate 3 and says "continue," Phase N+1 design begins immediately with the screen inventory already drafted instead of starting from scratch.

**When to skip phase pipelining:**
- Single-phase products
- Phase N+1 scope is marked "TBD" or "open" in the PRD
- The user said "pause" or "done" at Gate 3

---

## How to Pass Context to Parallel Agents

Each agent must be fully self-contained. Do not assume agents share memory.

Pass to each agent:
1. The exact file path(s) to read
2. The specific task to perform
3. The exact output file path to write to
4. Any relevant rules or framework files to follow

Do NOT pass:
- References to "what the other agent is doing"
- Shared intermediate state
- Instructions that depend on the other agent's output

---

## Merging Parallel Outputs

After a parallel block completes, the orchestrating agent (main Claude session) is responsible for:

1. Reading all output files
2. Identifying agreements, conflicts, and gaps
3. Producing a synthesis before the next sequential step begins
4. Surfacing any unresolved conflicts to the user at the next gate

Synthesis is always sequential. Never gate on raw parallel output alone.

---

## Sequential Steps — Never Parallelize These

| Step | Why Sequential |
|------|---------------|
| Research → PRD | PRD content depends on research findings |
| PRD → Reviews | Reviews need a complete PRD |
| Reviews → Apply Fixes | Fixes depend on synthesis of both reviews |
| Apply Fixes → Design | Design must use the corrected PRD |
| Design → Execute | Execute must use the approved design catalog |
| Execute → Validate | Can only validate what has been built |
| Validate → Ship | Ship gate requires clean validation |
| Ship → Learn | Learn report needs post-ship data |
