# Validate User Stories

A **semantic** content validator for the user-stories breakdown. Different from `/pipeline-doctor` (structural integrity) and from the Gate 3 quality checks (mechanical pass/fail rules). This command actually reads the AC content and flags duplicates, contradictions, vague language, broken FE/BE coherence, and sizing mismatches that are invisible to schema-level checks.

Use when:
- The breakdown is large (50+ stories) and no human is reading every AC carefully.
- After major edits via `/change-mode` (especially "Designs arrived" refreshes).
- Before Gate 3 approval — catches issues the gate quality checks miss.
- Before `/prd-to-jira` — catches vague AC that won't survive engineering review and prevents bad Jira tickets.
- After running `/validate-prd` and fixing PRD issues — the breakdown may now drift from the updated PRD.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

**Cost note:** This is the **most expensive** command in the skill. The user-stories file for a non-trivial feature can be 500KB+ (nestfully-ai is 614KB / 75 stories / 11,220 lines). Expect 1–3 minutes of runtime and significant token consumption. Run on-demand only.

---

## Step 0 — Input selection

Ask the PM:

```
Which feature's user-stories breakdown should I validate?

1. The most recently active feature — derived from the most recently modified
   _pipeline-state.json under ~/Desktop/Resources/PDLC Workflow Docs/.
2. A specific feature — give me the feature name.
3. A specific breakdown file — paste a file path.

(Default: 1)
```

Resolve to:
- `~/Desktop/Resources/PDLC Workflow Docs/[feature]/user-stories/[feature]-user-stories.md`
- Or the path the PM gave.

Read the breakdown. Also read **supporting context** (used by several checks):
- `prd/[feature]-prd.md` (for Story↔PRD traceability)
- `_pipeline-state.json` (for `user_stories.epics[]`, `user_stories.draft_stories[]`, `user_stories.mode`)
- `design/[feature]-phase-*-designs.md` if any exist (for design-dependency checks)

If the breakdown file is missing, stop. State: "No user-stories breakdown found for [feature]. Run `/user-stories` or `/build-product` first."

If the breakdown is very large (>500KB), tell the PM upfront: "This breakdown is [N]KB / [N] stories. The full validation will take ~[N] minutes. Continue? (yes / split into batches / cancel)"

---

## Step 1 — Run the nine checks

### Check 1: Story ↔ PRD traceability

Read the PRD's user-stories list (typically Section 7 — Phased Plan — with one user-story bullet per phase). Build a set of PRD-listed user stories (by short description).

Read the breakdown's per-story sections. Build a set of US-IDs with titles.

Cross-check:
- **PRD stories not in breakdown** (orphan PRD entries): every PRD user story should have at least one matching US-ID in the breakdown. List any missing.
- **Breakdown stories not in PRD** (orphan breakdown entries): every breakdown US-ID should have an antecedent in the PRD. List any that look fabricated or that wandered in via `/change-mode` without a corresponding PRD update.
- **FE/BE pair coverage**: if the breakdown splits a PRD story into a linked FE/BE pair (US-1.1 / US-1.2), both halves should exist. Flag pairs with only one half present.

Severity: **WARNING** for orphans in either direction. **CRITICAL** if a key PRD user story (one referenced by a success metric or NFR) has no breakdown coverage.

### Check 2: AC duplication / contradiction across stories

Extract every Gherkin `Scenario:` block from every story. For each scenario, normalize:
- Lowercase
- Strip whitespace
- Keep the `Given` / `When` / `Then` structure intact

Then look for:

**Duplicate scenarios:**
- Same `Given X / When Y / Then Z` triple appearing in two or more stories.
- Same scenario name (`Scenario: Saved searches empty state`) with substantively similar bodies in two stories.
- Same negative case (e.g., "invalid input rejected") restated identically in 5+ FE stories — suggests boilerplate copy-paste rather than story-specific AC.

For each duplicate group: list the US-IDs that share it, quote the scenario, and suggest whether to (a) factor it out to a shared "story 0 / pre-condition" doc, (b) keep it once on the canonical story and reference it from others, or (c) accept the duplication if it's genuinely story-specific.

**Contradictions:**
- Story A says "Then the user sees an empty state with [copy X]"; Story B says "Then the user sees an empty state with [copy Y]" for the same screen — contradiction.
- Story A says "Given the API returns 200"; Story B's BE counterpart says "the API returns 201 for create" — endpoint contract drift.
- Story A's AC implies user has permission; Story B's AC for the same role implies they don't.

Severity: **WARNING** for duplications (sometimes intentional). **CRITICAL** for contradictions (will surface as bugs in QA).

### Check 3: FE/BE pair coherence

For every story with `Linked pair: US-X.Y` in its header (or that appears in the Sequence Map's "Related To" column), open both halves of the pair and verify:

- **Data flow alignment**: If the FE story's AC says "Then the saved searches list renders the items returned from `/api/v1/saved-searches`", the BE story should define that endpoint's response shape with matching fields.
- **Error path alignment**: If the FE story handles `404 — no saved searches`, the BE story should specify when 404 is returned vs 200 with empty array. Or vice versa — FE handles empty array, BE returns empty array (not 404). Mismatched conventions are bugs.
- **Permission alignment**: If the BE story says "requires `saved_searches:read` permission", the FE story should reflect that (loading state for unauthorized, redirect, etc.).
- **Naming alignment**: Field names referenced by the FE in copy/labels should match field names in the BE response.

For each pair, run the four sub-checks and flag any mismatch.

Severity: **CRITICAL** for endpoint-contract / permission mismatches. **WARNING** for naming / convention mismatches.

### Check 4: AC specificity

For every story, scan the AC scenarios for vague language. Flag scenarios that:

- Use "it works" / "it succeeds" / "it functions correctly" as the `Then` clause without specifying observable behavior.
- Use "the user sees the correct result" without defining "correct".
- Use "fast response" / "quick load" without a numeric threshold.
- Use "appropriate error message" without specifying the message or behavior.
- Use "if needed" or "if applicable" — non-deterministic AC that QA can't verify.

For each flagged scenario, suggest concrete replacement language. Example:
- ❌ "Then the saved search appears in the list"
- ✅ "Then the saved search appears in the list with its title, query text, and last-modified timestamp; it's selectable; it persists across page reloads"

Severity: **WARNING** for vague AC. **INFO** for AC that has some vagueness but the surrounding scenarios clarify intent.

DRAFT stories are **exempt** from this check — they're known to have placeholder AC that will be refreshed when designs arrive.

### Check 5: Sizing sanity

For each story, compute:
- **Scenario count**: number of `Scenario:` blocks in the AC.
- **Distinct surface count**: how many distinct screens (for FE) or endpoints (for BE) does the AC reference?
- **Cross-boundary indicators**: does the AC reference multiple external systems, multiple data tables, or complex state transitions?

Compare to the size label (`S`, `M`, `L`, `L+`) using these rough heuristics:

| Size | Typical scenario count | Surface count | Complexity indicators |
|---|---|---|---|
| S | 1–3 | 1 | No cross-boundary; single data flow |
| M | 4–7 | 1 | Single screen / endpoint; may touch 2 systems |
| L | 8–12 | 2–3 | Multiple screens / endpoints OR significant state transitions |
| L+ | 13+ | 4+ | High complexity; PRD called it out as a hard problem; needs split |

Flag mismatches:
- **Under-sized**: marked S but has 10 scenarios across 3 screens.
- **Over-sized**: marked L but has 2 scenarios on a single screen.
- **L+ without a proposed split**: rule from the breakdown spec — L+ requires the per-story section to include a "Proposed split into US-X.Y + US-X.Z" note. Flag if missing.

Severity: **WARNING** for size mismatches. **INFO** for borderline cases (e.g., M vs L) that are judgment calls.

DRAFT stories' `*` sizing carries explicit uncertainty — sizing-sanity findings on DRAFT stories should be **INFO**, not WARNING.

### Check 6: DRAFT consistency

Read `_pipeline-state.json` → `user_stories.draft_stories[]` and `user_stories.mode`.

For each US-ID listed in `draft_stories[]`:
- The corresponding per-story section in the breakdown file should have `Status: ⚠ DRAFT — needs design` at the top.
- Sizing should end with `*` (e.g., `M*`, `L*`, `L+*`).
- There should be a `**Known design gaps:**` block in the story section.
- UX state coverage (empty/loading/error/populated) should NOT be required — its absence is expected.

Flag missing markers (a story listed in state as DRAFT but lacking the markers in the breakdown).

Conversely, scan all stories in the breakdown for any that have `Status: DRAFT` markers but are NOT in `user_stories.draft_stories[]` in state — that's state/file drift.

Severity: **WARNING** for inconsistency in either direction. **INFO** if the markers are present but slightly malformed (e.g., emoji missing).

### Check 7: Wave / dependency sanity

Read the breakdown's Build Sequence Map (the table at the top with US-ID / Depends On / Related To / Wave columns).

Check:
- **Acyclic dependencies**: No story depends on itself transitively. Build a dependency graph from the "Depends On" column and check for cycles.
- **Wave ordering**: If US-X is in Wave 3 and depends on US-Y, US-Y must be in Wave 1, 2, or 3 (not 4+). Stories can't depend on later-wave work.
- **Phase ordering**: If US-X is in Phase 1 and depends on US-Y in Phase 2, that's a phase-order violation (Phase 2 ships after Phase 1).
- **FE→BE dependency direction**: For linked pairs, the FE story usually depends on the BE story (FE can't render data it doesn't have). Flag pairs where the BE depends on the FE (unusual; possibly an error).
- **Wave size sanity**: If one wave has 40 stories and the next has 1, that's likely a planning artifact — surface as INFO.

Severity: **CRITICAL** for cycles or phase-order violations. **WARNING** for wave-order violations. **INFO** for wave-size imbalances.

### Check 8: Layout — meat-first / appendix structure

Per the breakdown spec (`ai-framework/06-user-stories.md` Step 5), the document is **meat-first**: front matter → at a glance → how to read → epic outlines → wave overview → per-story blueprints. Maintenance content (ID Stability Policy, refactor history, full per-story dependency table, format conventions, vendor-sprint detail) lives in appendices at the end.

Scan the document top-to-bottom and find the line number of the **first** `### US-` blueprint header (call it `meat_start`). For every `## ` (level-2) header above `meat_start`, flag any of the following:

- `## ID Stability Policy` (or any header containing "ID Stability")
- `## Refactor summary` (or any header containing "Refactor summary" / "Refactor lineage" / "Refactor authority")
- `## Build Sequence Map` if it contains the full per-story dependency table (a single-row-per-story table with `Depends on`, `Size`, `Type` columns). The small wave-overview table is fine in the meat; the big per-story table belongs in Appendix D.
- `## Format Conventions` / `## Story archetype reference`
- `## Pass 1 status` / `## Pass 2 status` / similar `## Pass N` markers
- Vendor sprint detail blocks at section level (e.g., `### REA delivery alignment` outside the wave-overview)

For each finding: report the line number, the header text, and the recommended appendix destination (A through F).

Severity: **WARNING** for each section in the wrong place. If the cumulative front-matter is >100 lines above `meat_start`, escalate the bundle to a single **CRITICAL** finding with the recommendation to refactor to appendix-style.

### Check 9: Count parity (prose claims vs actual blueprint count)

The number of unique `### US-` blueprint headers is the source of truth. Every prose claim must match.

1. **Compute the truth.** Count unique `### US-X.Y` blueprint headers in the document. This is `total_blueprint_count`. Count per-epic by scanning `### US-` headers under each `## Epic [N]:` section. This gives `epic_blueprint_count[N]`.
2. **Extract every numeric story-count claim from prose.** Search for patterns:
   - `Total v[N] stories.*\b(\d+)\b`
   - `\bTotal stories\b.*\b(\d+)\b`
   - `\b(\d+) v\d+ stories\b`
   - `\bAll (\d+) stories\b`
   - `\b(\d+) stories (written|done|remaining)\b`
   - Per-epic openers like `\b(\d+) stories\.` immediately following an Epic section header.
3. **Compare.** For each claim, report:
   - The line number and exact quoted claim.
   - The claimed number vs the actual count (total or per-epic as appropriate).
   - The delta.
4. **Per-epic claims** match against `epic_blueprint_count[N]` for the epic the claim appears under (look up by nearest preceding `## Epic [N]:` header).
5. **Cut stories** (with `Status: CUT` in their blueprint) are excluded from "active" counts but included in "written" counts. If the doc uses both framings, validate each against the appropriate basis.

For each mismatch: severity is **WARNING** if delta ≤ 5%, **CRITICAL** if delta > 5% or if the claim is in a prominent location (At-a-glance row, document title, executive summary line). Propose the corrected number; do not auto-edit until Step 3 walkthrough.

DRAFT-only counts (e.g., "Mode: DRAFT · 12 DRAFT stories") are validated against `user_stories.draft_stories[].length` in `_pipeline-state.json`, not the blueprint header count.

---

## Step 2 — Compose findings

Same structure as `/validate-prd`. Group by check, sort by severity within each check.

### Inline chat summary

```
━━━ Validate User Stories — [Feature Name] — [YYYY-MM-DD HH:MM] ━━━

Breakdown: ~/Desktop/Resources/PDLC Workflow Docs/[feature]/user-stories/[feature]-user-stories.md
Size: [N] KB · [N] stories ([N] FE, [N] BE) · last modified [date]
Mode: full | DRAFT | MIXED · [N] DRAFT stories

Check 1: Story ↔ PRD traceability    — [N] WARNING · [N] CRITICAL
Check 2: AC duplication / contradiction — [N] WARNING · [N] CRITICAL
Check 3: FE/BE pair coherence        — [N] WARNING · [N] CRITICAL
Check 4: AC specificity              — [N] WARNING · [N] INFO
Check 5: Sizing sanity               — [N] WARNING · [N] INFO
Check 6: DRAFT consistency           — [N] WARNING · [N] INFO
Check 7: Wave / dependency sanity    — [N] CRITICAL · [N] WARNING · [N] INFO
Check 8: Layout (meat-first / appendix) — [N] CRITICAL · [N] WARNING
Check 9: Count parity (prose ↔ blueprints) — [N] CRITICAL · [N] WARNING

Summary: [N] CRITICAL · [N] WARNING · [N] INFO

Full report: ~/Desktop/Resources/PDLC Workflow Docs/[feature]/validation/[feature]-validate-stories-[YYYY-MM-DD-HHMM].md
```

If zero findings: `✓ All 9 checks passed — breakdown is internally coherent.`

### Full markdown report

Path: `~/Desktop/Resources/PDLC Workflow Docs/[feature]/validation/[feature]-validate-stories-[YYYY-MM-DD-HHMM].md`

Create the `validation/` subfolder if it doesn't exist. (Shared with `/validate-prd` reports — both write under the same `validation/` folder.)

Same structure as the validate-prd report:
- Front matter (path, size, mode, draft count, last modified)
- Summary table per check
- Findings grouped by severity within each check
- Each finding has Where (US-ID + section + short quote), What, Suggested fix
- "Passed checks" list
- "How to act on this report" footer with concrete guidance

---

## Step 3 — Present and walk through findings

Print the inline summary to chat. Then ask:

```
Want me to walk through the [N] CRITICAL + WARNING findings and propose fixes one at a time?
(yes / no / just the CRITICAL ones / skip — write the report only)
```

If yes: for each finding (CRITICAL first), show:
- The finding from the report.
- A concrete proposed fix (with edit-ready text for AC tightening, sizing changes, missing DRAFT markers, etc.).
- Ask: "Apply this fix? (yes / skip / show me the story context first)"

Apply approved fixes:
- **AC specificity / wording fixes**: edit the breakdown file in place.
- **Sizing changes**: update the size column in the Sequence Map + the per-story header.
- **DRAFT marker fixes**: add/remove `Status: ⚠ DRAFT — needs design` markers per state.
- **Missing FE/BE half** (Check 1): do NOT auto-create — propose running `/user-stories` or `/change-mode` to add the missing story properly.
- **Endpoint contract mismatches** (Check 3): do NOT auto-fix — these are real architectural decisions; flag for PM + tech-lead resolution.
- **Dependency cycles** (Check 7): do NOT auto-fix — propose specific edge to break.
- **Layout findings** (Check 8): per-section, propose moving the section to the named appendix (A–F). Move the entire section verbatim — do not rewrite it. Update internal cross-references if any pointed at the old location. Apply only on PM approval.
- **Count parity** (Check 9): for each prose claim that drifted, edit the number in place to match the actual blueprint count (or the per-epic count for per-epic claims). Show before/after for each. Batch-apply on confirmation is acceptable for count fixes since the source-of-truth basis is unambiguous.

After applying fixes, append a "Fixes applied in this run" section to the report (same pattern as validate-prd).

---

## Step 4 — Write the report file

Always write the full markdown report to disk. Use the timestamped filename — never overwrite prior reports.

Update or create `~/Desktop/Resources/PDLC Workflow Docs/[feature]/validation/_validate-stories-history.md` with a newest-first one-line log:

```markdown
# Validate Stories History — [Feature Name]

- 2026-05-22 14:30 — 75 stories · 2 CRITICAL, 8 WARNING, 5 INFO — [report path]
- 2026-05-21 09:15 — 75 stories · 0 CRITICAL, 3 WARNING, 1 INFO — [report path]
```

---

## Rules

- **Read-only by default.** No edits to the breakdown until the PM approves a specific fix.
- **DRAFT stories are graded gently.** Checks 4 (AC specificity) and 5 (sizing) downgrade or skip findings on stories listed in `user_stories.draft_stories[]` — they're expected to have placeholder AC and uncertain sizing.
- **No external network calls.** Only reads local files.
- **Quote sparingly.** Findings should quote 5–20 words of the relevant AC line, not entire scenarios.
- **Don't invent issues.** Only flag what's clearly present. False positives erode trust at 75-story scale.
- **Idempotent.** Two runs back-to-back on the same breakdown produce identical findings.
- **Token budget for large breakdowns.** For breakdowns >500KB, process in batches: by Epic, by Phase, or by Wave. Don't truncate without telling the PM the analysis is partial.
- **Cross-reference state.** `user_stories.epics[]` and `user_stories.draft_stories[]` from `_pipeline-state.json` are the source of truth for epic assignments and DRAFT status. The breakdown file is the source of truth for AC content. Reconcile between them — that's where Check 6 (DRAFT consistency) catches state/file drift.
