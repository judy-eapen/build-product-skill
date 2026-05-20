# Parallel Execution Rules

These rules govern when and how to run parallel agents in the product development pipeline. Read this before implementing any step marked [PARALLEL] in `build-product.md`.

---

## Execution Model — Use the Agent Tool

**Every "spawn" instruction in this file means a real Agent tool call, not role-playing.**

Do NOT simulate parallel agents by switching personas in the same context window. That is not parallel execution — it is sequential execution with anchoring bias. The second "agent" always sees the first agent's output in conversation history, which corrupts the independence that parallel review exists to provide.

**The correct execution pattern for every parallel block:**

1. Compose a fully self-contained prompt for each agent (see "How to Write Agent Prompts" below).
2. Call the Agent tool once per agent. Where the framework says "spawn simultaneously," make all Agent tool calls in the same message so they run concurrently.
3. Wait for all agents to return results.
4. Synthesize in the main orchestrator context (never inside an agent).

**Agent prompts must be self-contained.** Each agent receives only what it needs for its specific task. Do NOT pass:
- References to what other agents are doing or have done
- Another agent's output or findings
- Shared intermediate state or conversation history summaries
- Instructions that depend on another agent's output

---

## How to Write Agent Prompts

Every agent prompt must include four sections:

```
ROLE: [One sentence describing who this agent is and what it evaluates]

INPUTS:
- [Exact file path to read]
- [Any additional file path, e.g. codebase review]

TASK: [Specific task description — what to evaluate, what verdict to produce]

OUTPUT:
- Write your findings to: [exact output file path]
- Format: [any specific format requirements]
- Do not read or reference any other agent's output.
```

Do not add "also consider what the other reviewer said" or any cross-agent reference. Each agent's prompt is a closed system.

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

### Block 1 — Dual Review (Steps 3a + 3b)

**Why parallel:** Product Review and CTO Review both read the PRD and write to separate files. Zero dependency. Running them in the same context causes the second reviewer to anchor on the first — defeating the purpose of independent review.

**How to execute:**

Call the Agent tool twice in the same message (so they run concurrently). Compose each prompt using the template above:

**Agent 1 — Product Reviewer prompt:**
```
ROLE: You are a senior product leader reviewing a PRD. Apply the Product Reviewer
persona exactly as defined in ai-framework/personas.md (read that file first).

INPUTS:
- PRD to review: [exact path to PRD file]

TASK: Review the PRD from a product lens. Cover document quality (structure, problem
definition, success metrics, scope, user stories, AC) and product quality (problem-solution
fit, workflows, edge cases, empty/error/loading states, onboarding, phasing).
Produce a verdict: Ready to lock / Needs minor edits / Needs significant work.
Include: Strengths, Gaps and blind spots, Improvements table, Quick wins, Edge cases,
Specific PRD suggestions.

OUTPUT:
- Write your review to: [product-review output path]
- Do not read or reference any other agent's output.
- Read ai-framework/rules.md and ai-framework/error-handling.md before executing.
```

**Agent 2 — Technical Reviewer prompt:**
```
ROLE: You are a senior technical leader reviewing a PRD. Apply the Technical Reviewer
persona exactly as defined in ai-framework/personas.md (read that file first).

INPUTS:
- PRD to review: [exact path to PRD file]
- Codebase review (Work pipeline only): [path, or omit if Personal pipeline]

TASK: Review the PRD from a technical lens. Cover architecture, data model, API contracts,
security, phasing, dependencies, technical risks, cost vs value.
If a codebase review is provided: assess whether the PRD is consistent with codebase
findings and whether HIGH risks from the codebase review are addressed in the PRD.
Produce a verdict: Ready to lock / Needs minor edits / Needs significant work.
Include: Strengths, Gaps and blind spots, Improvements table, Technical risks and
mitigations, Alternatives considered, Specific PRD suggestions.

OUTPUT:
- Write your review to: [technical-review output path]
- Do not read or reference any other agent's output.
- Read ai-framework/rules.md and ai-framework/error-handling.md before executing.
```

**Optional additional lenses:** If the PM activated optional review lenses at intake (Security, Accessibility, Data Privacy, AI Safety), spawn one additional Agent per activated lens in the same message, alongside the two default reviewers. Each additional agent receives only the PRD and its persona definition from `ai-framework/personas.md`. The synthesis step merges all reviewer outputs — each lens is treated as a single-source reviewer unless two lenses produce a conflicting recommendation.

**After all reviewers return — synthesize in the main orchestrator (not inside any agent):**
- Agreements (multiple reviewers raised the same issue): apply immediately, high confidence.
- Conflicts (directly contradictory recommendations): surface as conflict cards per `error-handling.md` Error Type 2. Never auto-resolve.
- Single-source findings (only one reviewer raised it): apply with a decision log note.

### Block 2 — Three-Way Validation (Steps 10a + 10b + 10c)

**Why parallel:** AC compliance, design match, and NFR checks each examine different dimensions of the same build. No ordering dependency. All three can read their inputs without waiting for each other.

**How to execute:**

Call the Agent tool three times in the same message (so they run concurrently):

**Agent 1 — AC Compliance prompt:**
```
ROLE: You are a QA engineer validating implementation completeness.

INPUTS:
- PRD: [exact path]
- Phase number: [N]

TASK: Check every acceptance criterion in the PRD for Phase [N]. For each criterion,
produce a verdict: PASS / PARTIAL / FAIL with one sentence of evidence.
Summarize: total criteria checked, pass count, partial count, fail count.

OUTPUT:
- Write your report to: [validation output path] — section "AC Compliance"
- Read ai-framework/rules.md before executing.
```

**Agent 2 — Design Match prompt:**
```
ROLE: You are a frontend engineer checking implementation fidelity against designs.

INPUTS:
- Design catalog: [exact path, or write "No design catalog — skip this check" if none exists]
- Phase number: [N]

TASK: Check the current UI implementation against the design catalog for Phase [N].
Note structural deviations only (not style preferences). Flag missing screens, wrong
navigation flows, missing states (empty/loading/error). If no design catalog exists,
write "Design Match: N/A — no catalog provided."

OUTPUT:
- Write your report to: [validation output path] — section "Design Match"
- Read ai-framework/rules.md before executing.
```

**Agent 3 — NFR Check prompt:**
```
ROLE: You are a platform engineer checking non-functional requirements.

INPUTS:
- PRD: [exact path]
- Phase number: [N]

TASK: Check non-functional requirements for Phase [N]. Specifically verify:
performance thresholds (if defined in PRD), error states present, loading states
present, empty states present, auth enforcement (if applicable), mobile layout
(if applicable), logging and observability. Verdict per item: PASS / FAIL / N/A.

OUTPUT:
- Write your report to: [validation output path] — section "NFR Check"
- Read ai-framework/rules.md before executing.
```

**After all three return — merge in the main orchestrator:**
- Combine the three sections into a single validation report at the output path.
- Overall result: Pass / Pass with notes / Fail.
- If any FAIL items exist: fix before proceeding to Gate 3.

### Block 3 — Parallel Export at Step 11 (Work pipeline)

**Why parallel:** Jira (always), Google Drive sync (optional), and Confluence publishing (optional) all read the same inputs and write to separate external systems. Zero dependency between them.

**Pre-flight (before calling agents):** Ask the PM which optional exporters to enable and collect required inputs (Drive folder, Confluence space) before the parallel block runs.

**How to execute:**

Call one Agent per enabled target in the same message (so they run concurrently):

**Agent 1 — Jira Export prompt:**
```
ROLE: You are creating Jira tickets from an approved user stories breakdown.

INPUTS:
- User stories breakdown: [exact path]
- PRD: [exact path]
- Jira project: [project key]

TASK: Read and follow subprompts/prd-to-jira.md exactly. Create one Jira ticket per
story. Write a manifest file mapping US-ID to Jira issue key.
If Jira MCP is unavailable, apply Error Type 4 from ai-framework/error-handling.md.

OUTPUT:
- Jira tickets created in project: [project key]
- Manifest: [jira-manifest output path]
- Read ai-framework/error-handling.md before executing.
```

**Agent 2 — Drive Sync prompt (only if enabled):**
```
ROLE: You are syncing a local feature folder to Google Drive.

INPUTS:
- Local feature folder: [exact path]
- Drive folder: [Drive folder URL or ID]

TASK: Read and follow ai-framework/07-drive-sync.md exactly. Mirror the feature folder
to Drive. Generate _FEATURE_SUMMARY.md at the feature folder root with quick links.
If Drive MCP is unavailable, skip cleanly and report.

OUTPUT:
- Synced to Drive folder: [Drive folder URL]
- Read ai-framework/error-handling.md before executing.
```

**Agent 3 — Confluence Publish prompt (only if enabled):**
```
ROLE: You are publishing a PRD to Confluence.

INPUTS:
- PRD: [exact path]
- Confluence space: [space key]
- Parent page (optional): [page title or ID]

TASK: Read and follow subprompts/prd-to-confluence.md exactly. Publish the PRD as a
Confluence page. Include quick-links to the Jira Epic and Drive folder when available.
If Confluence MCP is unavailable, apply Error Type 4 from ai-framework/error-handling.md.

OUTPUT:
- Published Confluence page URL
- Read ai-framework/error-handling.md before executing.
```

**After all enabled agents return — synthesize in the main orchestrator:**
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
