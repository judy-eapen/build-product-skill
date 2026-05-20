# Learn

Post-ship reflection. After a phase is deployed and users have interacted with it, capture what happened, what you learned, and what should change for the next phase.

## When to use

- After a phase has been shipped and used (even by just you and friends).
- Before starting the next phase's design or execution.
- When you want to decide: continue to the next phase, revise the PRD, or change direction.

---

## Your process

### 1. Gather context

Ask:

- Which PRD and Phase did you just ship? (e.g. `@~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`, Phase 1)
- How long has it been live? (days, weeks)
- Who has used it? (just you, friends, public)
- Any feedback you've already collected? (messages, observations, your own experience)

### 1b. Verify the feature is live

Verify the feature is live and behaving as expected using whatever monitoring or QA tool your team uses (browser for web apps, device or emulator for mobile, API client or logs for backend services, dashboards for data pipelines, etc.).

- Ask the user how they want to verify and the entry point (URL, endpoint, dashboard, app build).
- Check the key flows from the shipped phase for: crashes, broken outputs, missing data, error states, unexpected behavior.
- Note any issues found.

Include findings in the learning report under a **"Live Check"** section:

| Flow | Status | Notes |
|------|--------|-------|
| [Flow name] | Working / Broken / Partial | [What happened] |

If the feature cannot be verified in this session (no access, environment down, etc.), skip this step and note "No live check performed" in the report.

### 2. Check success metrics

Pull the success metrics from the PRD for this phase. For each:

| Metric | Target | Actual | Status | Notes |
|--------|--------|--------|--------|-------|
| [Metric from PRD] | [Target] | [Actual or estimate] | Hit / Partial / Missed / Unknown | [Context] |

If exact data isn't available from your analytics platform or reporting tool, use qualitative observations: "Used daily for a week" or "2 of 5 early users stopped after day 3."

### 3. Collect feedback themes

Organize feedback (from users, your own experience, or observations) into themes:

| Theme | Source | Frequency | Impact |
|-------|--------|-----------|--------|
| [What people said or experienced] | [Who] | [Once / Multiple / Everyone] | [High / Medium / Low] |

Examples of things to look for:
- What did people struggle with?
- What did people skip or ignore?
- What did people ask for that doesn't exist?
- What worked better than expected?
- What did you personally find annoying or missing?

### 4. Assess: what should change?

Based on metrics and feedback, decide:

- **Continue as planned:** Next phase proceeds as specified in the PRD. No changes needed.
- **Adjust next phase:** Minor tweaks to user stories or priorities for the next phase. Update the PRD.
- **Revise the PRD:** Significant changes needed (new features, removed features, changed scope). Run `/review-prd` again after changes.
- **Pivot or pause:** The core assumption isn't working. Revisit the research or problem statement.

### 5. Produce the learning report

```markdown
# Learning Report: [App Name] — Phase [N]

**Date:** [Date]
**Phase shipped:** [Date shipped]
**Time in use:** [Duration]
**Users:** [Who and how many]

## Success Metrics

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| ... | ... | ... | ... |

## Live Check

| Flow | Status | Notes |
|------|--------|-------|
| ... | ... | ... |

[Or: "No live check performed."]

## Feedback Themes

| Theme | Source | Frequency | Impact |
|-------|--------|-----------|--------|
| ... | ... | ... | ... |

## What Worked
- [Thing that went well and why]
- [...]

## What Didn't Work
- [Thing that didn't work and why]
- [...]

## Surprises
- [Something unexpected — good or bad]
- [...]

## Decision

**Next step:** [Continue as planned / Adjust next phase / Revise PRD / Pivot or pause]

**Changes to make:**
- [Specific change 1]
- [Specific change 2]
- [...]

## PRD Updates Needed
- [ ] [Specific PRD section to update, if any]
- [ ] [...]
```

### 6. Save

Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/learning/[feature-name]-phase-[N]-learning.md`.

---

## Important

- **Honest over positive.** If something didn't work, say so. The point is to learn, not to justify.
- **Qualitative is fine.** You don't need analytics dashboards. "Users stopped after day 2" or "the team flagged friction in step 3" is valid data.
- **Short and actionable.** This isn't a postmortem essay. Focus on what to *do* next.
- **Close the loop.** If PRD changes are needed, list them explicitly so the next step is clear.

---

## Knowledge Base Update

After the learning report is complete, append a structured entry to `~/Desktop/Resources/PDLC Workflow Docs/_knowledge-base.md`. If the file does not exist, create it.

The entry must include exactly five fields:

```markdown
## [Feature name] — [YYYY-MM-DD]

- **Top risk that materialized during build:** [one sentence]
- **Sizing accuracy:** [what was estimated versus what actually took, in plain English]
- **One decision that would have been made differently with hindsight:** [one sentence]
- **One pattern that worked well and should be repeated:** [one sentence]
```

Append the entry to the bottom of the knowledge base file. Never overwrite existing entries. The knowledge base lives at the root of the PDLC Workflow Docs folder, not inside any feature folder, so it persists across features.
