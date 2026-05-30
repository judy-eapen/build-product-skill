# User Stories Breakdown

Runs after Gate 2 (design approval) and before Gate 3 (breakdown approval) when called from the orchestrator, OR can be called standalone via `/user-stories` when a PM has an approved PRD and wants the breakdown without running the full pipeline.

Produces a standalone, human-readable user-stories document with exhaustive acceptance criteria, FE/BE pairing, testing notes, **multi-epic grouping**, and an optional **DRAFT mode** for stories whose design details aren't finalized yet. **AC format is not assumed** — Step 2.7 decides per feature (or per story) whether Gherkin or plain-English criteria reads clearest, and shows the PM a side-by-side sample before committing. This document becomes the source of truth for Jira ticket creation (Step 11 or standalone `/prd-to-jira`).

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

### Write a thorough Epic Description

For each proposed Epic, write a **plain-English Description** that explains what this Epic is for, written so a stakeholder who has not read the PRD can understand it. Required fields per Epic:

- **Title** — following intake naming convention if specified.
- **Theme** — one-line tag (e.g., `Saved Searches - Listing`). Used as a short label.
- **Description** — **3–6 sentences in plain English.** What this Epic covers, why it exists, who benefits, and how it fits into the broader feature. No jargon, no file paths, no "see PRD section X" references. A non-technical reader should finish this paragraph knowing what the Epic is for. Pull substance from PRD Section 2 (Scope), the PRD problem statement, and the specific stories in this Epic — do not generic-ify.

### Present the proposal to the PM

Show the proposed grouping for confirmation:

```
Proposed Epic grouping — [Feature Name]

EPIC 1: [Epic title following intake convention if specified]
  Phase: 1
  Theme: [one-line tag]
  Description:
    [3–6 sentences in plain English explaining what this Epic covers, why it exists,
    and who benefits. No file paths or "see PRD" references.]
  Stories ([N]):
    US-1.1 [FE] View saved searches
    US-1.2 [BE] Saved searches endpoint
    US-1.3 [FE] Delete a saved search
    ...

EPIC 2: [Epic title]
  Phase: 1
  Theme: [...]
  Description:
    [...]
  Stories ([N]):
    ...

EPIC 3: [Epic title]
  Phase: 2
  Theme: [...]
  Description:
    [...]
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

Produce a single table covering every story. The `Wave` column is computed in **Step 2.5** below — leave it blank in this initial pass; Step 2.5 fills it in.

| US-ID | Title | Type | Epic | Phase | Wave | Depends On | Related To | Size | DRAFT? |
|---|---|---|---|---|---|---|---|---|---|
| US-1.1 | View saved searches | FE | Epic 1 | 1 | W3 | US-1.2 | US-1.2 (BE pair) | M* | DRAFT |
| US-1.2 | Saved searches endpoint | BE | Epic 1 | 1 | W1 | — | US-1.1 (FE pair) | S | — |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |

Rules for the table:
- **Epic**: name from Step 1.5. Critical for downstream Jira export — every story must have an Epic assigned.
- **Wave**: computed in Step 2.5 (topological sort on `Depends On`). Format `W1`, `W2`, ... `W[N]`. Global numbering across the whole feature, not reset per phase or epic.
- **Depends On**: hard prerequisites. List the US-IDs of stories that must ship before this one can start. Use `—` for none.
- **Related To**: soft links, especially the FE/BE counterpart for the same feature. List the US-IDs. FE/BE pair links here do NOT count as dependencies — pair siblings often ship in the same wave.
- **Size**: S (less than a day), M (one to three days), L (more than three days or significant unknowns). If a story would be larger than L, mark `L+` and propose a split in the per-story section. **In DRAFT mode**, append `*` to indicate design-driven uncertainty (e.g., `M*`).
- **DRAFT?**: in full mode, this column is omitted. In DRAFT or MIXED mode, mark `DRAFT` for stories awaiting design refresh, or `—` for stories complete as-is.

Below the table, produce a one-paragraph **build-order summary** a PM could read aloud in sprint planning. (After Step 2.5 runs, the summary should reference wave numbers, not loose ordering.)

> "Wave 1 (foundation) ships US-1.2 BE auth and US-1.5 BE schema in parallel. Once W1 is done, Wave 2 unlocks US-2.1 + US-2.2. Critical convergence at Wave 6 — everything in W7+ depends on it. Launch gate at Wave [N]."

---

## Step 2.5 — Wave Sequencing

Compute a **wave assignment** for every story via topological sort on the `Depends On` column. Waves group stories that can ship in parallel; later waves depend on earlier ones.

### Algorithm

1. **Initialize** an empty assignment `wave[us_id]` for every story.
2. **Wave 1** = every story whose `Depends On` is `—` (no hard prerequisites). Set `wave[us_id] = 1` for each.
3. **Wave N** (for N = 2, 3, ...): every story whose `Depends On` US-IDs are all already assigned to waves `1..N-1`. Set `wave[us_id] = N` for each.
4. **Repeat** until every story has a wave.
5. **Cycle detection**: if at any pass no new story can be assigned (all remaining stories depend on each other or on unassigned stories), the dependency graph contains a cycle. Stop. Report which US-IDs are in the cycle and flag for PM resolution before continuing. Cycles are real bugs — surface them, don't paper over.

### Phase ordering constraint

A story's wave assignment is the **max** of:
- The wave computed from its dependencies (above), AND
- (Phase × 1.5)-rounded-up if the PRD's Phased Plan strictly orders the phases. (E.g., if Phase 2 cannot start until Phase 1 fully ships, the earliest Phase 2 wave is `(last Phase 1 wave) + 1`.)

If the decision log (`decisions/[feature]-decision-log.md`) explicitly allows phase parallelism, ignore the phase constraint and let dependencies alone drive wave assignment.

### FE/BE pair handling

A linked FE/BE pair (Related To, not Depends On) is **not** a hard dependency — the FE story does not block on the BE story being deployed. Pairs commonly land in the same wave (built in parallel, integrated at the end of the wave). The orchestrator should NOT add an FE→BE edge to the dependency graph based on the "Related To" column alone.

However, if the FE story's AC explicitly says "Given the BE endpoint US-X.Y is deployed", that IS a hard dependency — add it to `Depends On` and let the wave algorithm route accordingly.

### Wave Summary section

After computing waves, write a **Wave Summary** below the build-order summary in the breakdown file:

```markdown
## Wave Summary

| Wave | Theme | Stories | Critical? |
|---|---|---|---|
| W1 | Foundation (BE schema, auth, config) | US-1.2, US-1.5, US-2.3 (3 stories) | — |
| W2 | Data-layer APIs | US-1.3, US-1.4, US-1.6 (3 stories) | — |
| W3 | UI layer (initial screens) | US-1.1, US-1.7, US-1.8, US-1.9 (4 stories) | — |
| W4 | Notifications backend | US-2.1, US-2.2 (2 stories) | — |
| W5 | Notifications UI | US-2.4, US-2.5 (2 stories) | — |
| W6 | Cross-cutting search refinement | US-3.1, US-3.2, US-3.3 (3 stories) | **Critical convergence** — W7+ all depend on W6 |
| W7 | Saved-search-from-notification flow | US-3.4, US-3.5 (2 stories) | — |
| ... | ... | ... | ... |
| W12 | Launch gate | US-4.1 (release-blocking polish) | **Launch gate** |
```

Annotate two wave types:
- **Critical convergence wave** — a wave whose completion unlocks a large downstream block (e.g., 3+ subsequent waves all have at least one dependency on this wave). Mark it explicitly so the PM and team know it's the make-or-break point.
- **Launch gate wave** — the last wave before user-facing release. Usually contains polish stories, release-readiness checks, or feature flags. Mark it explicitly.

### Quality checks performed during wave sequencing

The orchestrator should surface these as Gate 3 quality-check findings (severity in brackets):

- **Cycles in dependency graph** [CRITICAL] — flag US-IDs in the cycle, refuse to proceed until PM resolves.
- **Story with no Wave assigned** [CRITICAL] — every story must have a wave (or the cycle check above caught it).
- **Wave imbalance** [INFO] — if one wave has 5x the stories of the median wave (e.g., 40 stories in W3 while W1, W2, W4 each have ~5), flag for PM review. Often indicates missing dependencies that would split that wave.
- **Phase straddling** [INFO] — a single wave contains stories from two different phases. Surface for PM awareness (sometimes intentional, sometimes a bug).

### Update the Sequence Map

After computing waves, go back to the Sequence Map table from Step 2 and fill in the **Wave** column for every story.

---

## Step 2.7 — Acceptance-criteria format fit-check (Gherkin vs. plain English)

**Do not assume Gherkin.** Before writing any acceptance criteria, decide whether `Scenario / Given / When / Then` is actually the clearest format for *this* feature's stories, or whether plain-English criteria would be easier for anyone reading the ticket to understand. This honors the workspace rule "Acceptance Criteria — Gherkin or plain bullet points, whichever is clearest."

### When Gherkin helps vs. when it gets in the way

- **Gherkin fits** behavioral, stateful, multi-path stories: FE flows with distinct states and triggers, BE endpoints with clear request → response → error paths, anything with several conditional branches QA must walk. The Given/When/Then structure earns its ceremony there.
- **Gherkin gets in the way** for: simple config or content/copy changes, data migrations or backfills, research / spike / investigation tickets, plumbing with a single obvious assertion, or any story where the behavior is one straight line. Forcing Given/When/Then onto these adds ceremony without clarity — a plain-English checklist reads better and tells the reader faster "what is this ticket supposed to do."

Assess the stories from the Sequence Map and classify the feature:
- **Gherkin** — most/all stories are behavioral and benefit from it.
- **Plain English** — most/all stories are simple/declarative; Gherkin would be noise.
- **Mixed** — some epics or stories want Gherkin, others want plain English (record which).

### Show the PM a side-by-side sample and let them choose

Pick **one representative story from this feature** (use its real Description and behavior — not a generic example) and write its acceptance criteria **both ways**, then present this block to the PM:

```
━━━ Acceptance-criteria format ━━━

For [feature], my read is: [Gherkin / Plain English / Mixed] — because [one-sentence reason
tied to the nature of these stories].

Here is one real story (US-[X.Y] — [title]) written both ways so you can compare readability:

── Option A: Gherkin ──
Scenario: [happy path]
  Given [...]
  When [...]
  Then [...]
Scenario: [negative/error]
  Given [...]
  Then [...]

── Option B: Plain English ──
This ticket is done when:
- [Plain, testable criterion — happy path, stated so anyone can check it]
- [Negative/validation criterion]
- [Edge criterion]
- [Error-handling criterion]

Both are testable. Gherkin is more structured for QA automation; plain English is faster
for any stakeholder to read and confirm "yes, that's what this ticket should do."

Which format should I use for this breakdown?
(a) Gherkin everywhere   (b) Plain English everywhere   (c) Mixed — I'll tell you which
[recommended: <your classification>]
━━━
```

Accept the PM's answer. If they pick **Mixed**, ask which epics/stories use which (or propose a split and let them confirm). Record the decision in `_pipeline-state.json`:

```json
"user_stories": {
  "ac_format": "gherkin | plain | mixed",
  "ac_format_overrides": { "Epic 2": "plain", "US-3.4": "gherkin" }
}
```

`ac_format` is the default for every story; `ac_format_overrides` (optional) names the exceptions when Mixed. Step 3 writes each story's AC in its resolved format. The choice carries through to validation, Gate 3, and Jira export (the AC custom field gets whatever format was chosen — never silently converted back to Gherkin).

If this step is re-run via `/change-mode` or a Gate 3 reopen and `ac_format` already exists in state, reuse it without re-asking unless the PM raises it.

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

**Description**

3–6 sentences in plain English explaining what this ticket is trying to do. Written for a stakeholder who has not read the PRD: what work is being done, who it serves, what behavior the user will see (or what the system will do, for BE stories), and how it fits into the broader Epic. No file paths, no "see PRD section X" references, no jargon. This is the field a designer, QA, or new engineer reads first to understand the ticket — make it self-contained. Distinct from the formal "User Story" below, which is the role-goal-benefit framing.

**User Story**

As a [role], I want [goal] so that [benefit].

**Acceptance Criteria** — write in the **format chosen at Step 2.7** for this story (`ac_format`, or its `ac_format_overrides` entry). Cover all applicable categories (happy / negative / edge / error) per the rules below, in whichever format. Write the number of criteria/scenarios this story actually needs — simple stories may have 2–3, complex stories may have 10+. Do not pad to match the template; do not cap at the template count.

*If the chosen format is **Gherkin**:*

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

*If the chosen format is **Plain English**:*

\`\`\`
This ticket is done when:
- [Happy path — the most common successful flow, stated as a testable check anyone can confirm]
- [Negative case — invalid input / missing required field / validation failure / unauthorized]
- [Edge case — empty state / max-min boundary / concurrent action / offline / locale-timezone]
- [Error case — upstream timeout / partial failure / retry / fallback]
\`\`\`

Each plain-English criterion must be **testable and specific** — state the condition and the observable outcome (e.g., "When the search returns no results, the page shows the empty-state message and a 'clear filters' action"), not a vague intention ("search works well"). Plain English changes only the syntax, not the rigor: the same four categories and the same UX-state coverage rules below still apply.

**Testing Notes (high level)**
- **Test-coverage areas:** [list, e.g. UI rendering, data flow, error handling, accessibility, performance]
- **Cross-boundary verification:** how to verify this works end-to-end with the [linked pair / dependency]. Example: "Hit the BE endpoint US-1.2 directly with a saved-search payload; confirm the FE in US-1.1 renders it correctly."
- **Edge cases to cover in QA:** [list]
- **Data conditions to test:** [list, e.g. empty, single item, max payload size, special characters, unicode]
- **Performance threshold:** [pull from PRD NFR section if applicable]
- **HIGH risks from codebase review affecting this story:** [list any, with mitigation note]
- **UX state coverage (FE stories only):** empty / loading / error / populated — confirm at least one scenario/criterion covers each
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

**Description**

3–6 sentences in plain English explaining what this ticket is trying to do. Same rules as full-mode Description (stakeholder-readable, no file paths, no PRD section references). Acknowledge any design-driven ambiguity in plain English ("exact UI copy and screen flow will be pinned down once designs land"), but still describe the intent of the ticket fully.

**User Story**

As a [role], I want [goal] so that [benefit].

**Acceptance Criteria — best effort from PRD** (in the Step 2.7 chosen format for this story)

*Gherkin:*

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

*Plain English:*

\`\`\`
This ticket is done when:
- [Happy path — behavioral only, no design-specific copy/labels]
- [Negative or error case where the PRD has enough detail to write it]
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

### Acceptance-criteria coverage rules (apply to both formats)

These rules govern **what the AC must cover**, regardless of whether the story is written in Gherkin or plain English (per Step 2.7). "Scenario" below means a Gherkin `Scenario:` block *or* a plain-English criterion bullet — the coverage bar is identical.

**Count scales with story complexity. There is no fixed number.** A simple story (e.g., "log analytics event when user clicks Save") may need 2–3 criteria. A complex story (e.g., "submit a 12-field form with cross-field validation, three error paths, and a draft autosave") may need 10 or more. Do not pad to match the template, and do not cap at the template's example count.

Each non-DRAFT story must cover these AC categories — write each category exhaustively for this specific story (some categories may need multiple criteria, others just one):

- **Happy path** — at least 1. The most common successful flow. Add more if the story has multiple legitimate happy paths (e.g., two user roles each with a different happy path).
- **Negative cases** — invalid input, missing required fields, validation failures, unauthorized access (where applicable). At least 1, more if the story has multiple distinct validation rules.
- **Edge cases** — empty state / first use, max boundary, min boundary, concurrent actions, offline or degraded service, timezone or locale if applicable. At least 1, more if multiple boundaries apply.
- **Error cases** — upstream timeouts, partial failures, retries, fallbacks. At least 1 (especially for BE stories), more if the story integrates with multiple upstream systems each with distinct failure modes.

**Skip a category only if it genuinely doesn't apply** (e.g., a read-only stats display has no "negative input" case). If you skip a category, the story is exempt from that one. Do not skip just to keep the count low.

DRAFT stories aim for happy path + at least one behavioral negative/error criterion when the PRD supports it. Edge cases tied to UX states (empty / loading) are deferred to design refresh.

**Syntax by format:**
- *Gherkin:* each scenario starts with `Scenario:`, followed by `Given` / `When` / `Then` / `And` lines, indented two spaces.
- *Plain English:* a `This ticket is done when:` lead-in followed by one testable bullet per criterion — each stating the condition and the observable outcome.

For BE stories: criteria describe API behavior (request → response), not UI.
For FE stories: criteria describe user-visible behavior (action → screen state).

### UX state coverage for FE stories

For every **non-DRAFT** FE story, confirm at least one scenario or criterion covers each of:
- **Empty state** — what the user sees when there's no data yet.
- **Loading state** — what the user sees while waiting.
- **Error state** — what the user sees when something goes wrong.
- **Populated state** — what the user sees in the normal "data present" case.

If any state is missing, add a scenario/criterion for it (in the story's chosen format).

For DRAFT FE stories, UX state coverage is **not required** — these are tracked as known gaps in `user_stories.draft_stories[]` and refreshed via `/change-mode` later.

---

## Step 4 — Self-check (pre-Gate-3)

Before presenting the breakdown to the PM, run the quality checks:

1. **Every PRD user story appears in the breakdown** (no drops). Cross-check against the PRD's Phased Plan.
2. **Every story has a unique US-ID.** No duplicates.
3. **Every story is assigned to exactly one Epic** from the Step 1.5 grouping.
3a. **Every story has a thorough Description** (3–6 sentences, plain English, no file paths, no "see PRD section X" placeholders).
3b. **Every Epic has a Description** (3–6 sentences, plain English) captured in `_pipeline-state.json` → `user_stories.epics[].description`.
4. **Every story has a Wave assignment** from Step 2.5 (W1, W2, ...). No story should be missing a wave (would indicate a cycle or an algorithm bug).
5. **No cycles in the dependency graph.** Step 2.5 detects cycles; surface them here as CRITICAL findings with the US-IDs involved.
6. **Every non-DRAFT story has at least 2 AC scenarios/criteria** (in the story's chosen format — Gherkin scenarios or plain-English criteria). Single-criterion non-DRAFT stories are insufficient. (DRAFT stories aim for ≥1 — count is reported but not required.)
7. **Every non-DRAFT story has at least one edge-case OR error-state scenario/criterion.** Not just happy path. (DRAFT stories exempt.)
7a. **AC format matches the Step 2.7 decision.** Every story's AC is written in the format recorded in `_pipeline-state.json` → `user_stories.ac_format` (honoring any `ac_format_overrides`). No story silently mixes formats against the decision.
8. **Every linked FE/BE pair has both sides present.** If you list US-1.1 (FE) → linked to US-1.2 (BE), US-1.2 must exist as its own story.
9. **No story is sized L+ without a proposed split.** If you marked L+, the per-story section must include a "Proposed split into US-X.Y + US-X.Z" note. (Applies to DRAFT stories too — sized as L+* gets the same treatment.)
10. **HIGH risks from the codebase review appear in at least one story's testing notes.** Traceability check. Read the codebase review file and confirm.
11. **UX state coverage per non-DRAFT FE story.** Empty / loading / error / populated — all four states have at least one scenario between them. DRAFT FE stories are exempt and counted as known gaps.
12. **Layout is meat-first.** No `## ID Stability Policy`, `## Refactor summary`, `## Per-story sequence table`, or `## Format Conventions` section appears in the document **above** the first `### US-` blueprint header. These belong in Appendix A–F at the end. If they appear above the meat, the layout is wrong — flag as WARNING and propose moving them to appendices.
13. **Count parity.** The number of unique `### US-` blueprint headers in the document is the source of truth. Every prose claim of the form "Total [N] stories", "[N] v1 stories", "All [N] stories written", or per-epic "[N] stories" must match this computed count. Run the parity check at write time: extract every numeric story-count claim, compare to the actual blueprint count (or per-epic blueprint count for per-epic claims), and either auto-correct or surface as WARNING. Hand-narrated counts that drift are the exact failure mode this check exists to prevent.

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

Structure — **meat-first, operational metadata in appendices**. Stakeholders (sponsors, designers, engineers, QA) open this doc to find what each story does and how it sequences. They should not have to scroll past hundreds of lines of refactor lineage, ID-stability policy, or full dependency tables to get there. Maintenance/audit content goes in appendices at the end.

**Top of the document (the meat):**

1. **Front matter — a strict allowlist, ≤ 8 lines.** Only these fields may appear, each one line. Nothing else goes in the front matter:
   - **Source PRD:** `prd/[feature]-prd.md` (vX.Y)
   - **Generated:** [date]
   - **Phases covered:** [list]
   - **Mode:** full / DRAFT / MIXED
   - **Stories:** [count, computed from blueprint headers] ([N] DRAFT)
   - **Waves:** [total wave count]
   - **Change history:** see [../changelog/[feature]-changelog.md](...) → "User Stories Breakdown" (the single pointer, same as Appendix B)
   - *(optional)* **Jira:** [N] tickets in project [KEY] — see [manifest](../jira-export/[feature]-jira-manifest.md) — a **compact one-liner**, not an issue-range narrative

   **Forbidden in the front matter (this is the junk that accretes over `/change-mode` runs — never write any of it):**
   - A **`Status:` line carrying a version + change narrative** (e.g. "Status: Final. v1.6 (date) — readability reformat. Every story restructured…"). A bare `Status: Draft` / `Status: Final` word is allowed; an attached version/refactor story is not.
   - **`Last updated:`**, **`Predecessor:`**, **`Refactor authority:`**, **`Refactor lineage:`**, **`Version history:`**, or any other bold-label line that narrates versions, prior versions, or what changed.
   - Multi-version prose of any kind. The version number lives in the title line; the full history lives in the centralized changelog (the one-line pointer above is the only nod to it).

   These forbidden fields are change-history by another name — the same rule as `## Change log` sections in `ai-framework/style-preferences.md` § Artifact Conventions, just in front-matter disguise. (The per-story `Status: ⚠ DRAFT — needs design` marker inside a blueprint, and gate-reopen `STATUS: DRAFT` banners, are different and stay — they describe live state, not version history.)
2. **At a glance** table (Total stories, Epics, Waves, Format, Notable dependencies). Counts derive from the single source-of-truth count (see "Source of truth for counts" rule below).
3. **How to read this document** — three-row table mapping audience (Executive sponsor / Engineer / Designer) to where to start.
4. **Epic outlines** — one block per Epic with a 2–3 sentence "What this delivers" outcome paragraph written for non-engineers. Pulled from `user_stories.epics[].description` in `_pipeline-state.json`.
5. **Build Sequence Map — wave overview** (the small table: Wave / Story count / Theme / Vendor dependency). The full per-story dependency table goes to Appendix D, not here.
6. **━━━ Per-story blueprints ━━━** — the canonical content. Per-story sections in US-ID order, grouped under `## Epic [N]: [name]` headers. This is where the document spends most of its lines and where stakeholders actually live.

**Appendices (end of document — maintenance and audit only):**

- **Appendix A — ID Stability Policy.** Verbatim from the workspace-level policy in `~/CLAUDE.md` plus any feature-specific notes. Required when the doc has been through any `/change-mode` insert/append, because it documents why letter-suffix IDs exist.
- **Appendix B — Change-history pointer (one line only).** Do **not** embed refactor history / changelog content in this document — not at the top, not in an appendix. Per `ai-framework/style-preferences.md` § Artifact Conventions, all version history lives in the centralized `changelog/[feature]-changelog.md` under its `## User Stories Breakdown` section. Appendix B is a single pointer line: `**Change history:** see [../changelog/[feature]-changelog.md](../changelog/[feature]-changelog.md) → "User Stories Breakdown" section.` On every `/change-mode` run, the per-version structural changes (Structural / Format / Sequencing / Dependency / Cuts) are written to that section of the changelog, never appended here.
- **Appendix C — Format conventions.** Story blueprint template, adversarial story format if Gherkin is retained for any epic, archetype reference.
- **Appendix D — Per-story sequence + dependency table.** The full multi-column table (US-ID / Title / Epic / Wave / Type / Skill / Depends on / Size) for every story. Useful for sequencing review; not what a sponsor or designer opens the doc to read.
- **Appendix E — Vendor / external dependency alignment** (if applicable). Per-vendor sprint mapping, contingency flags, mock-vs-live cutover plan.
- **Appendix F — PRD user-story cross-reference.** List of PRD user stories ↔ breakdown US-IDs (for the no-drops check).

**Source of truth for counts.** The story count is computed once, at write time, from the number of `### US-` blueprint headers in the body. Every other place that mentions a count ("At a glance", "Total v1 stories", per-epic counts in epic outlines, footer claims) reads from this single computed value. **Never hand-narrate counts in prose.** When `/change-mode` adds or cuts stories, it recomputes from blueprint headers and updates every claim site in the same pass. Per-epic counts are derived from the count of `### US-` headers under each `## Epic [N]:` section, not narrated.

**What does NOT go at the top.** Specifically: ID Stability Policy, full per-story sequence table, Format conventions / story archetype reference, REA-style vendor sprint detail, Pass 1 / Pass 2 status markers. All of these are maintenance content and live in appendices. **Refactor history / changelog content does not go in this document at all** — not at the top, not in an appendix — only the one-line Appendix B pointer to the centralized changelog. The Nestfully-AI v1.2 doc, which accreted 460 lines of front-matter before the first per-story blueprint, is the anti-pattern this rule exists to prevent.

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
      "theme": "one-line tag",
      "description": "3–6 sentence plain-English explainer captured at Step 1.5. Stakeholder-readable; used verbatim as the Summary block of the Jira Epic description.",
      "story_ids": ["US-1.1", "US-1.2", "US-1.3"]
    },
    {
      "epic_id": "E2",
      "title": "Saved Searches - Management",
      "phase": 1,
      "theme": "...",
      "description": "...",
      "story_ids": ["US-1.4", "US-1.5"]
    }
  ],
  "waves": [
    { "wave": 1, "theme": "Foundation (BE schema, auth, config)", "story_ids": ["US-1.2", "US-1.5", "US-2.3"], "critical_convergence": false, "launch_gate": false },
    { "wave": 2, "theme": "Data-layer APIs",                       "story_ids": ["US-1.3", "US-1.4"],            "critical_convergence": false, "launch_gate": false },
    { "wave": 3, "theme": "UI layer (initial screens)",            "story_ids": ["US-1.1", "US-1.7"],            "critical_convergence": false, "launch_gate": false },
    { "wave": 6, "theme": "Cross-cutting search refinement",       "story_ids": ["US-3.1", "US-3.2"],            "critical_convergence": true,  "launch_gate": false },
    { "wave": 12, "theme": "Launch gate",                          "story_ids": ["US-4.1"],                      "critical_convergence": false, "launch_gate": true }
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
- **Every story must have a thorough Description** (3–6 plain-English sentences) above the formal User Story. Stories without a Description are invalid. The Description is what flows into the Jira ticket's Description field at export.
- **Every Epic must have a Description** (3–6 plain-English sentences) captured at Step 1.5 and persisted in `_pipeline-state.json` → `user_stories.epics[].description`. Epics without a Description are invalid. The Description flows into the Jira Epic's Summary block at export.
- **Every story must have a Wave assignment** from Step 2.5. Cycles in the dependency graph are CRITICAL findings — surface them, do not paper over.
- **FE/BE pair (Related To) is not a hard dependency** unless the FE story's AC explicitly requires the BE deployed first. Pairs commonly land in the same wave.
- **DRAFT mode is opt-in.** Default is full mode (designs required). DRAFT mode requires explicit PM choice at Step 0.5.
- **Verbatim language from Voice of Customer (Step 1 of pipeline) should appear** in user-story narratives and AC where possible. Don't paraphrase user-facing language unless necessary.
- **Update `_pipeline-state.json`** at the end of this step. Persist `user_stories.epics[]`, `user_stories.draft_stories[]`, and `user_stories.waves[]` (one entry per wave with theme, story_ids, and any critical-convergence / launch-gate annotations). These fields are required by downstream steps (Jira export, `/change-mode`, `/validate-user-stories`, `/timeline`).
- **Self-check (Error Type 3 in error-handling.md):** does this breakdown contradict any decision recorded in the decision log (`decisions/[feature]-decision-log.md`)? If yes, flag to the PM before writing.
