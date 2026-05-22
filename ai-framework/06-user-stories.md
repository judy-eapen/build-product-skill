# User Stories Breakdown

Runs after Gate 2 (design approval) and before Gate 3 (breakdown approval) when called from the orchestrator, OR can be called standalone via `/user-stories` when a PM has an approved PRD and wants the Gherkin breakdown without running the full pipeline.

Produces a standalone, human-readable user-stories document with exhaustive Gherkin AC, FE/BE pairing, testing notes, **multi-epic grouping**, and an optional **DRAFT mode** for stories whose design details aren't finalized yet. This document becomes the source of truth for Jira ticket creation (Step 11 or standalone `/prd-to-jira`).

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 0 — Input Check (gracefully handle standalone calls)

Before doing anything else, determine which inputs you have available.

**Required input:**
- A PRD with user stories.

**Optional inputs (each improves the breakdown — warn about limitations if missing):**
- Design catalog — informs UX state coverage per FE story (empty / loading / error / populated).
- Codebase review — feeds HIGH-risk traceability into Testing Notes per story.
- Visual diagram — feeds screen-to-story mapping.

### If running inside the orchestrator (called from `/build-product`)

All inputs are already in conversation context at:
- PRD: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`
- Design catalog: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`
- Codebase review: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/codebase-review/[feature-name]-codebase-review.md`
- Visual diagram: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md`

Skip to Step 0.5.

### If running standalone (called via `/user-stories`)

Ask the PM:

> "To produce the user stories breakdown I need a PRD at minimum, plus some optional inputs that improve the output.
>
> **Required:**
> - Paste the PRD or give me the file path.
>
> **Optional (which do you have?):**
> - Design catalog — if yes, paste it or give me the path. (Without it, UX state coverage in the breakdown will be inferred from the PRD only — may miss design-driven detail.)
> - Codebase review — if yes, paste or give path. (Without it, the breakdown won't link HIGH risks to stories in the Testing Notes.)
> - Visual diagram — if yes, paste or give path. (Without it, screen-to-story mapping will be inferred from the PRD only.)
>
> Tell me what you have and I'll work with that."

Wait for the PM's response. Also ask for the feature name (used for output file path).

For each optional input the PM doesn't have, explicitly note the limitation in the breakdown output. Example header note: "⚠ This breakdown was generated without a design catalog. UX state coverage was inferred from the PRD only."

If the PM does not provide a PRD, stop. State: "I need at least a PRD to produce a user stories breakdown. Please paste one or run /create-prd first."

---

## Step 0.5 — Design availability check (chooses DRAFT mode vs. full mode)

Detect whether a finalized design catalog exists. Look for any file matching:

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-*-designs.md
```

**If at least one design-catalog file exists** AND Gate 2 has been approved in `_pipeline-state.json` → full mode is the default. Skip ahead and confirm with the PM:

```
Designs are finalized for this feature (Gate 2 approved). I'll write stories with full
UX state coverage per FE story. Proceed?
(yes / I want to write some in DRAFT mode anyway / cancel)
```

**If no design catalog exists** OR Gate 2 is not approved, ask the PM explicitly:

```
I don't see finalized designs for this feature yet. How do you want to proceed?

1. Wait — pause here. I'll proceed when you've finalized designs and re-run /build-product
   (or /user-stories standalone).

2. Write user stories now using the PRD only. Design-dependent stories will be marked
   "Status: DRAFT — needs design" at the top of the story. UX state coverage (empty /
   loading / error / populated) will not be required for DRAFT stories — they'll be
   flagged as known gaps at Gate 3 rather than failures. Sizing will be marked with *
   (e.g., M*) for low-confidence rows where design uncertainty is the dominant factor.

   When designs arrive later, run /change-mode → "Designs arrived" trigger. It will
   walk through every DRAFT story with the new designs in context and refresh AC,
   sizing, and UX state coverage in one pass — and update the Jira tickets in place.

3. Cancel — I'll come back to this later.
```

Wait for the PM's choice.

- **Choice 1 (Wait):** Stop. Print: "Paused. Run `/build-product` (Fast or Gated mode) when designs are ready." Update `_pipeline-state.json` → `current_step` to indicate the pause. Exit cleanly.
- **Choice 2 (Write anyway, DRAFT mode):** Proceed in **DRAFT mode**. Persist `_pipeline-state.json` → `user_stories.mode = "DRAFT"` and continue.
- **Choice 3 (Cancel):** Stop with no state changes.

If the PM chose full mode but a subset of phases is missing designs (e.g., Phase 1 has designs, Phase 2 doesn't), set `mode = "MIXED"` and write Phase 1 stories in full mode + Phase 2 stories in DRAFT mode. Report this to the PM at the end of the step.

---

## Step 1 — Read inputs and extract stories

Read the PRD's Phased Plan (Section 7). Extract every user story across every phase.

For each PRD user story, decide whether it is:
- **FE only** — UI work, no backend changes.
- **BE only** — backend / API / data work, no UI changes.
- **Both** — UI work plus backend work. In this case, **split into a linked FE/BE pair** with the same root number (e.g., the PRD story "user sees their saved searches" becomes US-1.1 FE + US-1.2 BE).

If the PRD already labeled stories FE / BE, respect that labeling.

Pull the design catalog (if available) to confirm which screens map to which stories. If a FE story has no design counterpart in full mode, flag it as "missing design" in the breakdown.

Pull the codebase review to identify which HIGH risks touch which stories. Carry that into the Testing Notes section per story.

### Determine which stories are DRAFT

In DRAFT mode (or MIXED mode), classify each story:

- **DRAFT** — story has FE work whose AC depends on screen-level design detail (e.g., copy, exact field labels, empty/loading/error states, screen flow order) that is not in the PRD. Examples: any FE story for a screen with no corresponding design file.
- **Not DRAFT** — story is BE-only, OR story is FE but the AC can be written confidently from the PRD alone (e.g., a "log analytics event when user clicks save" story whose AC is purely behavioral).

Record the DRAFT classification per story. This drives the per-story format in Step 3 and the state-file tagging in Step 5.

---

## Step 1.5 — Propose epic grouping (multi-epic support)

After classifying stories but before writing the Sequence Map, propose an **epic grouping**: how the stories should be organized into Jira Epics.

### Default grouping heuristic

1. **One Epic per PRD phase**, unless a phase has clear functional sub-clusters.
2. **Functional clusters within a phase** — if a phase contains stories addressing visibly distinct functional areas (e.g., one cluster for "Search," another for "Filters," another for "Saved Searches"), propose a sub-epic per cluster. Detect clusters by looking at the PRD's Phased Plan groupings, story titles, and any sub-headers within the phase section.
3. **Single epic for the whole feature** — only if the PRD has one phase AND fewer than ~10 stories AND no obvious functional clusters.

Use the PM's intake convention for Epic naming if specified at intake. Read `_pipeline-state.json` → `intake.jira_ticket_conventions` for any epic-title format guidance (e.g., `"Feature - Sub-feature"`).

### Present the proposal to the PM

Show the proposed grouping for confirmation:

```
Proposed Epic grouping — [Feature Name]

EPIC 1: [Epic title following intake convention if specified]
  Phase: 1
  Theme: [one-line description of what unifies these stories]
  Stories ([N]):
    US-1.1 [FE] View saved searches
    US-1.2 [BE] Saved searches endpoint
    US-1.3 [FE] Delete a saved search
    ...

EPIC 2: [Epic title]
  Phase: 1
  Theme: [...]
  Stories ([N]):
    ...

EPIC 3: [Epic title]
  Phase: 2
  Theme: [...]
  Stories ([N]):
    ...

Total: [N] epics, [N] stories.

Adjust anything?
  • "Merge Epic 1 and Epic 2" — combine into one
  • "Split Epic 3 — move US-3.4 and US-3.5 into a new epic" — break out a subset
  • "Rename Epic 2 to 'X'" — change the title
  • "Move US-1.3 to Epic 2" — reassign a story
  • "Single epic" — collapse everything into one
  • "Looks good" — accept and proceed

What needs to change?
```

Wait for the PM's response. Apply any adjustments and re-present the grouping. Loop until the PM accepts.

If the PM intake convention requires `"Feature - Sub-feature"` Epic titles, propose titles in that format (e.g., `"Saved Searches - Listing"`, `"Saved Searches - Management"`). If the intake is silent on epic naming, use plain descriptive titles.

Record the final grouping for later use in Step 2 (Sequence Map column) and Step 5 (state persistence).

---

## Step 2 — Build the Sequence Map

Produce a single table covering every story:

| US-ID | Title | Type | Epic | Phase | Depends On | Related To | Size | DRAFT? |
|---|---|---|---|---|---|---|---|---|
| US-1.1 | View saved searches | FE | Epic 1 | 1 | US-1.2 | US-1.2 (BE pair) | M* | DRAFT |
| US-1.2 | Saved searches endpoint | BE | Epic 1 | 1 | — | US-1.1 (FE pair) | S | — |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |

Rules for the table:
- **Epic**: name from Step 1.5. Critical for downstream Jira export — every story must have an Epic assigned.
- **Depends On**: hard prerequisites. List the US-IDs of stories that must ship before this one can start. Use `—` for none.
- **Related To**: soft links, especially the FE/BE counterpart for the same feature. List the US-IDs.
- **Size**: S (less than a day), M (one to three days), L (more than three days or significant unknowns). If a story would be larger than L, mark `L+` and propose a split in the per-story section. **In DRAFT mode**, append `*` to indicate design-driven uncertainty (e.g., `M*`).
- **DRAFT?**: in full mode, this column is omitted. In DRAFT or MIXED mode, mark `DRAFT` for stories awaiting design refresh, or `—` for stories complete as-is.

Below the table, produce a one-paragraph **build-order summary** a PM could read aloud in sprint planning:

> "Start with US-1.2 BE and US-1.1 FE in parallel (linked pair). Once US-1.2 is deployed, US-2.1 and US-2.2 can begin. Final block is US-3.1 + US-3.2 which depend on the auth changes in Phase 2."

---

## Step 3 — Per-story sections

### Apply intake-captured title conventions

Before composing any story title, read `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/_pipeline-state.json` → `intake.jira_ticket_conventions`. If the PM specified a title format (e.g., "verb-first, `[BE]`/`[FE]` prefix"), apply it to every story title here. Examples:

- Convention: "verb-first, `[BE]`/`[FE]` prefix" → BE story title becomes `[BE] Create notes endpoint`, FE story title becomes `[FE] Add note from listing card`.
- Convention: "BE and FE always separate" → never produce a single story labeled `Type: both`; split into a BE story and an FE story linked as a pair.

If `intake.jira_ticket_conventions` is empty or silent on title format, use plain descriptive titles without prefixes.

### Full-mode story format (default)

For each US-ID in the Sequence Map that is **not DRAFT**, write a section with this structure:

```markdown
### US-[X.Y] — [Title]

**Epic:** [Epic name from Step 1.5]
**Type:** FE | BE | both
**Phase:** [N]
**Linked pair:** US-[A.B] ([type]) — [one-line note on relationship], or "None"
**Depends on:** [US-IDs or "None"]
**Size:** S | M | L | L+ (proposed split: ...)

**User Story**

As a [role], I want [goal] so that [benefit].

**Acceptance Criteria (Gherkin)**

Write the number of scenarios this story actually needs — simple stories may have 2–3, complex stories may have 10+. Do not pad to match the template; do not cap at the template count. Cover all four categories (happy / negative / edge / error) per the rules below.

\`\`\`
Scenario: [Happy-path — the most common successful flow]
  Given [...]
  When [...]
  Then [...]

# Then add additional scenarios as the AC requires.
# Category cues (write 1 or more per applicable category — do not copy verbatim):

Scenario: [Negative case — e.g., invalid input, missing required field, validation failure, unauthorized]
  ...

Scenario: [Edge case — e.g., empty state, max/min boundary, concurrent action, offline, locale/timezone]
  ...

Scenario: [Error case — e.g., upstream timeout, partial failure, retry, fallback]
  ...
\`\`\`

**Testing Notes (high level)**
- **Test-coverage areas:** [list, e.g. UI rendering, data flow, error handling, accessibility, performance]
- **Cross-boundary verification:** how to verify this works end-to-end with the [linked pair / dependency]. Example: "Hit the BE endpoint US-1.2 directly with a saved-search payload; confirm the FE in US-1.1 renders it correctly."
- **Edge cases to cover in QA:** [list]
- **Data conditions to test:** [list, e.g. empty, single item, max payload size, special characters, unicode]
- **Performance threshold:** [pull from PRD NFR section if applicable]
- **HIGH risks from codebase review affecting this story:** [list any, with mitigation note]
- **UX state coverage (FE stories only):** empty / loading / error / populated — confirm at least one scenario covers each
```

### DRAFT-mode story format

For each US-ID in the Sequence Map that **is DRAFT**, write a section with this structure:

```markdown
### US-[X.Y] — [Title]

**Status:** ⚠ DRAFT — needs design. Refresh via `/change-mode` → "Designs arrived" when finalized.
**Epic:** [Epic name from Step 1.5]
**Type:** FE | BE | both
**Phase:** [N]
**Linked pair:** US-[A.B] ([type]) — [one-line note on relationship], or "None"
**Depends on:** [US-IDs or "None"]
**Size:** S* | M* | L* | L+* — sizing carries design-driven uncertainty; revisit at refresh.

**User Story**

As a [role], I want [goal] so that [benefit].

**Acceptance Criteria (Gherkin) — best effort from PRD**

\`\`\`
Scenario: [Happy-path scenario name — behavioral only, no design-specific copy/labels]
  Given [...]
  When [...]
  Then [...]

Scenario: [Negative or error case where the PRD has enough detail to write it]
  Given [...]
  When [...]
  Then [...]
\`\`\`

**Known design gaps (refresh required):**
- Exact copy for empty / loading / error states
- Specific field labels and order
- Screen flow if multiple steps are involved
- [Anything else the PRD doesn't pin down]

**Testing Notes (high level)**
- **Test-coverage areas:** [behavior-focused; UI rendering noted as "TBD per design"]
- **Cross-boundary verification:** how to verify this works end-to-end with the [linked pair / dependency].
- **Edge cases to cover in QA:** [list — behavioral only]
- **Data conditions to test:** [list]
- **Performance threshold:** [pull from PRD NFR section if applicable]
- **HIGH risks from codebase review affecting this story:** [list any]
- **UX state coverage:** ⚠ Pending design refresh.
```

### Gherkin coverage rules

**Scenario count scales with story complexity. There is no fixed number.** A simple story (e.g., "log analytics event when user clicks Save") may need 2–3 scenarios. A complex story (e.g., "submit a 12-field form with cross-field validation, three error paths, and a draft autosave") may need 10 or more. Do not pad scenarios to match the template, and do not cap at the template's example count. The template above shows one happy-path scenario plus category cues — fill in the actual scenarios this specific story requires.

Each non-DRAFT story must cover these AC categories — write each category exhaustively for this specific story (some categories may need multiple scenarios, others may need just one, depending on the story):

- **Happy path** — at least 1 scenario. The most common successful flow. Add more if the story has multiple legitimate happy paths (e.g., a feature with two user roles each having a different happy path).
- **Negative cases** — invalid input, missing required fields, validation failures, unauthorized access (where applicable). At least 1 scenario, more if the story has multiple distinct validation rules each worth specifying.
- **Edge cases** — empty state / first use, max boundary, min boundary, concurrent actions, offline or degraded service, timezone or locale if applicable. At least 1 scenario, more if multiple boundaries apply.
- **Error cases** — upstream timeouts, partial failures, retries, fallbacks. At least 1 scenario (especially for BE stories), more if the story integrates with multiple upstream systems each with distinct failure modes.

**Skip a category only if it genuinely doesn't apply** (e.g., a read-only stats display has no "negative input" case). If you skip a category, the story is exempt from that one. Do not skip just to keep the count low.

DRAFT stories aim for happy path + at least one behavioral negative/error scenario when the PRD supports it. Edge cases tied to UX states (empty / loading) are deferred to design refresh.

Use proper Gherkin syntax: each scenario starts with `Scenario:`, followed by `Given` / `When` / `Then` / `And` lines. Indent two spaces.

For BE stories: scenarios describe API behavior (request → response), not UI.
For FE stories: scenarios describe user-visible behavior (action → screen state).

### UX state coverage for FE stories

For every **non-DRAFT** FE story, confirm at least one scenario covers each of:
- **Empty state** — what the user sees when there's no data yet.
- **Loading state** — what the user sees while waiting.
- **Error state** — what the user sees when something goes wrong.
- **Populated state** — what the user sees in the normal "data present" case.

If any state is missing, add a scenario for it.

For DRAFT FE stories, UX state coverage is **not required** — these are tracked as known gaps in `user_stories.draft_stories[]` and refreshed via `/change-mode` later.

---

## Step 4 — Self-check (pre-Gate-3)

Before presenting the breakdown to the PM, run the quality checks:

1. **Every PRD user story appears in the breakdown** (no drops). Cross-check against the PRD's Phased Plan.
2. **Every story has a unique US-ID.** No duplicates.
3. **Every story is assigned to exactly one Epic** from the Step 1.5 grouping.
4. **Every non-DRAFT story has at least 2 Gherkin scenarios.** Single-scenario non-DRAFT stories are insufficient. (DRAFT stories aim for ≥1 scenario — count is reported but not required.)
5. **Every non-DRAFT story has at least one edge-case OR error-state scenario.** Not just happy path. (DRAFT stories exempt.)
6. **Every linked FE/BE pair has both sides present.** If you list US-1.1 (FE) → linked to US-1.2 (BE), US-1.2 must exist as its own story.
7. **No story is sized L+ without a proposed split.** If you marked L+, the per-story section must include a "Proposed split into US-X.Y + US-X.Z" note. (Applies to DRAFT stories too — sized as L+* gets the same treatment.)
8. **HIGH risks from the codebase review appear in at least one story's testing notes.** Traceability check. Read the codebase review file and confirm.
9. **UX state coverage per non-DRAFT FE story.** Empty / loading / error / populated — all four states have at least one scenario between them. DRAFT FE stories are exempt and counted as known gaps.

If any check fails, flag as a WARNING (severity: WARNING) for Gate 3. Do not silently auto-fix. Surface to the PM at Gate 3 so they can decide to fix-first or approve-anyway.

Additionally, if any stories are DRAFT, surface a Gate 3 information block:

```
ℹ DRAFT stories pending design refresh: [N]
   Epic 1: US-1.1, US-1.3
   Epic 2: US-2.4
   ...
   
   These will not block Gate 3 approval. After Jira tickets are created, they will be
   tagged with a "draft" label so they can be found later. Run /change-mode → "Designs
   arrived" once designs are finalized to refresh each one in place.
```

---

## Step 5 — Write the file and update state

Write the complete document to:

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md
```

Structure:
1. Front matter (Source PRD, Generated date, Phases covered, Mode: full / DRAFT / MIXED, DRAFT story count if any).
2. Epic grouping table from Step 1.5.
3. Build Sequence Map table.
4. Build-order summary paragraph.
5. Per-story sections in US-ID order, grouped under an `## Epic [N]: [name]` header per epic.
6. Appendix: list of PRD user stories cross-referenced to breakdown US-IDs (for the no-drops check).

### Update `_pipeline-state.json`

Persist the following under `user_stories`:

```json
"user_stories": {
  "mode": "full | DRAFT | MIXED",
  "path": "user-stories/[feature]-user-stories.md",
  "epics": [
    {
      "epic_id": "E1",
      "title": "Saved Searches - Listing",
      "phase": 1,
      "theme": "one-line description",
      "story_ids": ["US-1.1", "US-1.2", "US-1.3"]
    },
    {
      "epic_id": "E2",
      "title": "Saved Searches - Management",
      "phase": 1,
      "theme": "...",
      "story_ids": ["US-1.4", "US-1.5"]
    }
  ],
  "draft_stories": [
    { "us_id": "US-1.1", "epic_id": "E1", "reason": "no design catalog for Phase 1 yet" },
    { "us_id": "US-1.3", "epic_id": "E1", "reason": "no design catalog for Phase 1 yet" }
  ]
}
```

`draft_stories` is empty `[]` in full mode. `/change-mode` → "Designs arrived" reads this list to know which stories need refresh.

---

## Step 6 — Present Gate 3

After writing, present the breakdown to the PM at Gate 3 (defined in SKILL.md). The Gate 3 quality check block lists any flags from Step 4, plus the DRAFT-story information block if applicable. The PM can:

- **Approve** — proceeds to Step 11 (Jira Export).
- **Fix specific issues** — name them; the prompt revises the breakdown; re-presents Gate 3.
- **Reject and rework** — flag concerns; pipeline pauses.

Until Gate 3 is approved, do not proceed to Jira Export.

---

## Rules

- **Source of truth is the PRD.** Do not add user stories the PRD doesn't have. Do not invent acceptance criteria not implied by the PRD's AC.
- **Split into FE/BE pairs only when warranted.** A pure-UI story (e.g., a static info screen) doesn't need a BE pair.
- **Every story must be assigned to exactly one Epic** from the Step 1.5 grouping. Stories without an Epic are invalid.
- **DRAFT mode is opt-in.** Default is full mode (designs required). DRAFT mode requires explicit PM choice at Step 0.5.
- **Verbatim language from Voice of Customer (Step 1 of pipeline) should appear** in user-story narratives and AC where possible. Don't paraphrase user-facing language unless necessary.
- **Update `_pipeline-state.json`** at the end of this step. The `epics[]` and `draft_stories[]` fields are required by downstream steps (Jira export, `/change-mode`).
- **Self-check (Error Type 3 in error-handling.md):** does this breakdown contradict any decision recorded in the PRD's decision log? If yes, flag to the PM before writing.
