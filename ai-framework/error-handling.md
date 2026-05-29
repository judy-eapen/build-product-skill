# Error Handling Rules

These rules apply to every step in the pipeline. Inherited globally alongside `rules.md`. Every step must read both `rules.md` and `error-handling.md` before executing.

---

## Error Type 1 — Bad Output That Passed a Gate

If a PM realizes after approving a gate that the output is wrong, the following commands are available:

- `/reopen-gate-1`
- `/reopen-gate-2`
- `/reopen-gate-3`

When triggered:

1. Put the pipeline back into the pre-gate state.
2. Flag all downstream artifacts as `DRAFT` (add a header note to each affected file: `STATUS: DRAFT — gate reopened on [date]`).
3. Re-run the affected steps between the prior gate (or pipeline start, for Gate 1) and the reopened gate.
4. Re-present the gate with a full quality check.

Do not delete downstream artifacts. They remain as DRAFT until the gate is re-approved or they are explicitly overwritten by the re-run.

---

## Error Type 2 — Contradictory Parallel Agent Outputs

When the synthesis step finds a direct contradiction between agents that cannot be auto-resolved, produce a structured conflict card.

Each conflict card has exactly three fields:

```
CONFLICT [N]:
- Agent 1 ([name]) said: [statement]
- Agent 2 ([name]) said: [statement]
- Why they conflict: [one sentence]
```

The PM resolves it explicitly. The resolution is appended to the decision log sidecar (`decisions/[feature]-decision-log.md`, not the PRD body) before the pipeline continues.

Never auto-resolve a conflict by picking one side. Never silently merge conflicting recommendations.

---

## Error Type 3 — Step Output Contradicts the PRD

Any step that produces output must run a self-check before writing.

Self-check question: does this output contradict any decision already recorded in the decision log (`decisions/[feature]-decision-log.md`) or the PRD?

If yes:
1. Flag it to the PM before writing.
2. Show what the PRD / decision log says and what the new output says.
3. Ask the PM how to resolve: update the PRD, change the output, or stop and escalate.

Do not silently overwrite.

---

## Error Type 4 — External System Failure

Step 11 has up to three external export agents running in parallel (Jira always, Drive optional, Confluence optional). Each one handles failure independently — there is no global rollback.

### Jira export fails

1. Write the intended ticket content to the jira-export subfolder as a local markdown file:
   ```
   ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-export.md
   ```
2. Notify the PM that the Jira push failed but the content is preserved locally.
3. Include the original error message.
4. State: "The PM can retry the export without rerunning the pipeline."

If only some tickets fail (mid-batch), write the failed tickets to the local file and report which succeeded vs failed. The successful tickets remain in Jira.

### Drive sync fails (or MCP unavailable)

1. Notify the PM that Drive sync was skipped. Include the original error or "Drive MCP not available."
2. The local feature folder remains intact — it's the source of truth. No data is lost.
3. State: "Run `/drive-sync` later when the MCP is available."
4. Do not block the rest of Step 11. Jira and Confluence agents continue.

### Confluence publishing fails (or MCP unavailable)

1. Write the intended Confluence page content to a local fallback:
   ```
   ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-confluence-export.md
   ```
2. Notify the PM. Include the original error.
3. State: "The PM can retry by running `/publish-to-confluence` standalone, or publish manually from the fallback file."
4. Do not block Jira or Drive agents.

### General rules

- Do not retry external pushes silently. Do not loop on external failures.
- Per-agent failure does NOT cascade. The three Step 11 agents are independent.
- The orchestrator's Step 11 summary reports per-target success/failure clearly.

---

## Inheritance

Every step in the pipeline must read this file alongside `rules.md` before executing. The four error types above apply universally.
