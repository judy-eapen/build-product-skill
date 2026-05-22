# Validate PRD

A **semantic** content validator for the PRD. Different from `/pipeline-doctor` (which checks structural integrity — schema, file existence, gate states). This command actually reads the PRD's content and flags contradictions, hallucinated claims, vague language, placeholder text, and drift from the underlying research.

Use when:
- The PRD is large and you can't carefully re-read every section by hand.
- You've iterated through reviews + fixes + design updates and want to confirm the PRD still hangs together internally.
- Before Gate 1 approval (catches issues the gate quality checks miss).
- Before `/publish-to-confluence` (so stakeholders don't see a PRD with `[TBD]` left in).
- Before `/prd-to-jira` (catches vague AC that won't survive engineering review).

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

**Cost note:** This is an LLM-driven semantic analysis. On a PRD around 100KB (typical for a Work-pipeline feature), expect 30–60 seconds of runtime and a moderate token consumption. Larger PRDs (the nestfully-ai PRD is ~150KB) take ~1–2 minutes. Run on-demand, not auto-run at every gate.

---

## Step 0 — Input selection

Ask the PM:

```
Which feature's PRD should I validate?

1. The most recently active feature — derived from the most recently modified
   _pipeline-state.json under ~/Desktop/Resources/PDLC Workflow Docs/.
2. A specific feature — give me the feature name.
3. A specific PRD file — paste a file path.

(Default: 1)
```

Resolve the PRD path to one of:
- `~/Desktop/Resources/PDLC Workflow Docs/[feature]/prd/[feature]-prd.md`
- The path the PM gave.

Read the PRD. Also read **supporting context** (used by several checks below):
- `research/[feature]-research.md` (for VOC traceability)
- `codebase-review/[feature]-codebase-review.md` (for technical-grounding checks)
- `prd/decision-log.md` if it exists separately, otherwise the PRD's Section 10 decision log
- `_pipeline-state.json` (for `intake.*` and `gates.*` context)

If the PRD file is missing, stop. State: "No PRD found for [feature]. Run `/create-prd` first."

---

## Step 1 — Run the six checks

Each finding has a severity (CRITICAL / WARNING / INFO), a category, and a concrete fix suggestion. Findings are kept in a list; the report is composed at Step 2.

### Check 1: Internal consistency

Scan the PRD for cross-section contradictions. Look specifically for:

- **Section 4 (Data Model) vs Section 5 (API Contracts)** — does the data model contain the fields the API claims to return? Does the API expose data the model doesn't track?
- **Section 7 (Phased Plan) vs Section 2 (Scope)** — are all in-scope items represented in at least one phase? Are any phases delivering out-of-scope items?
- **Section 7 phase dependencies** — does Phase 2 reference a capability Phase 1 doesn't ship? Does any phase reference an external system that another phase introduces later?
- **Decision Log (Section 10) vs other sections** — does any locked decision contradict the body of the PRD? E.g., "Decision: we will not support guest checkout" but Section 3 (Roles & Permissions) lists guest users.
- **Roles & Permissions (Section 3) vs every other section** — are roles referenced in API contracts, data model, and user stories actually defined in Section 3? Are there roles in Section 3 that no other section uses?
- **Section 8 (Observability/NFR) vs Section 5 (API)** — are latency / error-rate targets in NFR aligned with the API endpoints they apply to? Or are they generic ("API should be fast") with no per-endpoint specificity?

For each contradiction, capture: the two sections in conflict, the specific text from each, and a one-sentence suggested resolution (PM still decides).

Severity: **WARNING** by default. Bump to **CRITICAL** if the contradiction would block engineering work (e.g., API endpoint references a field the data model doesn't have).

### Check 2: Hallucinated data

Scan for numeric claims, statistics, and competitor/market references. Flag any that:

- Don't have a citation or source reference (e.g., "increases conversion 30%" with no baseline study).
- Reference research / a study / a metric without saying where it came from.
- Reference user counts, market sizes, or competitor performance numbers without a corresponding research output.

Cross-check against the research output file (`research/[feature]-research.md`):
- If the PRD claims something that's NOT in the research file, that's likely hallucinated.
- If the PRD claims something that IS in the research file but the research file itself has no source, that's a pre-existing problem to surface.

Severity: **WARNING** for unsourced claims. **CRITICAL** if the claim is the core justification for a phase or feature decision.

### Check 3: Completeness

Search the PRD text for placeholders and gaps:

- Literal `[TBD]`, `[TODO]`, `[FIXME]`, `(tbd)`, `(todo)`, `<placeholder>`, `XXX`, `???` markers.
- Sections that are required by the PRD template but empty or contain only "[content here]" / "Not applicable" / one-sentence placeholders that should have detail.
- Required sections per the intake (`_pipeline-state.json` → `intake.permission_model` = yes implies Section 3 must be populated; `intake.backend_api_surface` = yes implies Section 5 must be populated).
- Decision log entries without dates or without resolution (open question disguised as a logged decision).

Severity: **CRITICAL** for required-section gaps. **WARNING** for `[TBD]` markers. **INFO** for "Not applicable" sections that the intake confirmed as not needed.

### Check 4: Voice-of-Customer (VOC) traceability

Read the research output file (`research/[feature]-research.md`). Extract any verbatim user quotes, pain-point language, or distinct phrasings the research captured.

Then scan the PRD for the same language in:
- Section 1 (Executive Summary) — should reflect the actual problem in user-language.
- User stories section — "As a [role], I want [goal]" narratives.
- AC scenarios where user-facing copy is described.

Flag if:
- The PRD uses generic AI-generated phrasing where the research captured specific user language ("users want better notifications" vs. the research quote: "I miss the listings I cared about because the email digest comes at 4am").
- The PRD invents quotes or attributes statements to users without research grounding.
- The PRD's user stories don't match the personas/segments the research identified.

Severity: **INFO** for stylistic drift (PRD paraphrased when verbatim was available). **WARNING** if the PRD invents user-attributed claims that aren't in research.

### Check 5: NFR measurability

Scan Section 8 (Observability & Non-Functional Requirements). For each NFR, check that it's:

- **Specific**: names the concrete metric (latency, error rate, throughput, availability target, etc.).
- **Measurable**: has a concrete threshold (e.g., "p95 latency < 200ms" not "fast response time").
- **Bounded**: applies to a defined scope (specific endpoint / page / data path).

Flag NFRs that are:
- Generic aspirations ("fast", "reliable", "scalable") without thresholds.
- Vague scopes ("the API should be performant") instead of per-endpoint.
- Missing a target value entirely.

Also check Section 9 (Testing Notes) — same rule: testing approaches should be specific to this feature, not boilerplate ("test happy path and edge cases" is too generic).

Severity: **WARNING** for vague NFRs. **INFO** for NFRs that have a threshold but no scope.

### Check 6: Scope coherence

Read Section 2 (In Scope and Out of Scope) carefully.

Flag:
- Items in "In Scope" that are also in "Out of Scope" (direct contradiction).
- Items in "Out of Scope" that are referenced as features in other sections (creeping scope).
- "In Scope" items that have no corresponding user stories / phase entries.
- Success metrics (Section 1) that depend on capabilities that are "Out of Scope".
- Decision log entries that re-introduce things explicitly marked "Out of Scope" without a clear reversal note.

Severity: **CRITICAL** for direct in-scope/out-of-scope contradictions. **WARNING** for scope-creep references. **INFO** for missing-story coverage of in-scope items.

---

## Step 2 — Compose findings

Group findings by check. Within each check, sort by severity (CRITICAL → WARNING → INFO).

Build two outputs: **inline chat summary** + **full markdown report**.

### Inline chat summary

```
━━━ Validate PRD — [Feature Name] — [YYYY-MM-DD HH:MM] ━━━

PRD: ~/Desktop/Resources/PDLC Workflow Docs/[feature]/prd/[feature]-prd.md
PRD size: [N] KB · [N] sections · last modified [date]

Check 1: Internal consistency        — [N] WARNING · [N] CRITICAL
Check 2: Hallucinated data           — [N] WARNING · [N] CRITICAL
Check 3: Completeness                — [N] CRITICAL · [N] WARNING
Check 4: VOC traceability            — [N] WARNING · [N] INFO
Check 5: NFR measurability           — [N] WARNING · [N] INFO
Check 6: Scope coherence             — [N] CRITICAL · [N] WARNING · [N] INFO

Summary: [N] CRITICAL · [N] WARNING · [N] INFO

Full report: ~/Desktop/Resources/PDLC Workflow Docs/[feature]/validation/[feature]-validate-prd-[YYYY-MM-DD-HHMM].md
```

If there are zero findings: replace the per-check counts with `✓ Clean` and the summary line with `✓ All 6 checks passed — PRD looks coherent.`

### Full markdown report

Path: `~/Desktop/Resources/PDLC Workflow Docs/[feature]/validation/[feature]-validate-prd-[YYYY-MM-DD-HHMM].md`

Create the `validation/` subfolder if it doesn't exist.

Structure:

```markdown
# Validate PRD — [Feature Name]

**Generated:** [YYYY-MM-DD HH:MM]
**PRD path:** `prd/[feature]-prd.md`
**PRD size:** [N] KB · [N] sections · last modified [date]
**Skill version:** v2.9.0+

## Summary

| Check | CRITICAL | WARNING | INFO |
|---|---|---|---|
| 1. Internal consistency | N | N | N |
| 2. Hallucinated data | N | N | N |
| 3. Completeness | N | N | N |
| 4. VOC traceability | N | N | N |
| 5. NFR measurability | N | N | N |
| 6. Scope coherence | N | N | N |
| **Total** | **N** | **N** | **N** |

## Findings

### CRITICAL

#### [Check #] — [Finding title]
- **Where in PRD:** [Section + paragraph reference, with a short quote]
- **What:** [Precise description of the contradiction / hallucination / etc.]
- **Suggested fix:** [Concrete suggested edit, with the proposed new wording where possible]

### WARNING

[Same structure]

### INFO

[Same structure, may be terse]

## Passed checks

[List of checks with no findings, brief.]

## How to act on this report

- **CRITICAL items** must be addressed before Gate 1 / before publishing to Confluence / before Jira export. They indicate the PRD has internal problems that will surface in downstream work.
- **WARNING items** should be reviewed. They may not block immediate work but indicate drift between intent and content.
- **INFO items** are diagnostic — read for context, act if relevant.
- Apply approved fixes via `/change-mode` for any change that touches downstream artifacts (designs, user stories, tickets). Direct edits to the PRD file are fine for typos / phrasing / NFR tightening.
- Re-run `/validate-prd` after applying fixes to confirm.
```

---

## Step 3 — Present and walk through findings

Print the inline summary to chat. Then ask:

```
Want me to walk through the [N] CRITICAL + WARNING findings and propose fixes one at a time?
(yes / no / just the CRITICAL ones / skip — write the report only)
```

If yes: for each finding (CRITICAL first), show:
- The full finding text from the report.
- A concrete proposed fix.
- Ask: "Apply this fix? (yes / skip / show me the PRD context first)"

Apply approved fixes:
- **Phrasing / NFR / completeness fixes**: edit the PRD file directly with the proposed text.
- **Decision-log entries needed**: add the entry to the decision log (or PRD Section 10).
- **Anything that contradicts a downstream artifact** (user stories, designs, tickets): do NOT auto-apply — propose running `/change-mode` instead, since the change has blast radius beyond the PRD.

After applying fixes, append a one-line note to the bottom of the report:

```markdown
## Fixes applied in this run

- [HH:MM] CRITICAL #3 — Added Decision Log entry for "guest checkout not supported"
- [HH:MM] WARNING #7 — Tightened NFR for /search endpoint: p95 < 250ms
- [HH:MM] WARNING #2 — Removed unsourced "30% conversion" claim from Section 1
```

Do not apply fixes silently — always confirm per-finding.

---

## Step 4 — Write the report file

Always write the full markdown report to disk, even if the PM declined the fix walkthrough. The report is the historical record. Use the timestamped filename — never overwrite prior reports.

Update or create `~/Desktop/Resources/PDLC Workflow Docs/[feature]/validation/_validate-prd-history.md`:

```markdown
# Validate PRD History — [Feature Name]

- 2026-05-22 14:30 — 1 CRITICAL, 3 WARNING, 2 INFO — [report path]
- 2026-05-21 09:15 — 0 CRITICAL, 1 WARNING, 0 INFO — [report path]
```

Newest entry at the top.

---

## Rules

- **Read-only by default** for the PRD until the PM explicitly approves a fix.
- **Reads multiple files** — the PRD, research, codebase review, decision log, state — to perform cross-checks. Don't make claims that depend on a file you didn't open.
- **No external calls.** Only local files. The validator doesn't query Jira, Figma, or any MCP.
- **Use the PRD's own structure.** Section numbers in findings should match the actual PRD section numbering (sometimes it's "Section 1" or "1. Executive Summary" or "## Executive Summary" — match the PRD's convention).
- **Quote sparingly.** Findings should quote a short fragment (5–20 words) of the contradicting text — not paragraphs. The full text is in the PRD; the report shouldn't duplicate it.
- **Don't invent contradictions.** Only flag contradictions that are clearly present in the text. When in doubt, downgrade severity or mark as INFO. False positives erode trust.
- **Idempotent.** Two runs back-to-back on the same PRD produce identical findings (modulo timestamps).
- **Token budget.** A 100KB PRD + 50KB research + 50KB codebase review can be read in one pass. Beyond that, batch by section. Do not truncate without surfacing that the analysis is partial.
