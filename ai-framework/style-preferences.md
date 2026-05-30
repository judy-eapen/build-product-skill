# Style Preferences

These are optional writing style preferences. Edit this file to match your own preferences. The skill will read from this file and apply whatever rules are here. Leave the file empty to use no style rules.

---

## Writing Style

- No em dashes.
- Bullet points over paragraphs.
- No filler phrases ("great question", "happy to help", "absolutely", "just", "very").
- Direct and rational tone — challenge weak reasoning.
- State assumptions explicitly.
- No invented facts, APIs, or behaviors.

## Artifact Conventions

Reader-facing artifacts (the PRD, system design, user-stories breakdown, etc.) are read by sponsors, engineers, designers, and QA. **Process-tracking content — what changed between versions, and the running log of decisions — reads like internal plumbing to those audiences and does not belong inside the artifact.** It lives in dedicated sidecar documents, and the artifact carries only a one-line pointer.

### No inline change history

- **No inline version-history in artifacts.** Do not embed any of the following inside an artifact file, at the top or anywhere:
  - `**v0.X (date):** ...` "what changed in this version" bullets or callouts.
  - A section titled **`## Change log`**, **`## Changelog`**, **`## Revision history`**, **`## Refactor history`**, or **`## Refactor summary`** — by any wording. A changelog is a changelog regardless of its heading; none of them go in a reader artifact.
  - "Document version", "Last updated", or "Generated [date]" stamp lines (a version number in the title line is the only version marker allowed).
  - **Front-matter version/refactor fields — the same junk in disguise.** A `**Status:**` line that carries a version + change narrative (e.g. "Status: Final. v1.6 (date) — readability reformat. Every story restructured…"), or any of **`**Predecessor:**`**, **`**Refactor authority:**`**, **`**Refactor lineage:**`**, **`**Version history:**`**, **`**Last updated:**`**. These read as front matter but are change history; they belong in the centralized changelog, not the doc. (A bare `Status: Draft`/`Status: Final` word, a per-story `Status: ⚠ DRAFT — needs design` marker, and gate-reopen `STATUS: DRAFT` banners are live-state markers, not version history — those stay.)
  - **Front matter is a tight, fixed allowlist.** Keep it to the few fields a reader needs to orient (what this is, what it was generated from, current counts/mode) plus the one-line changelog pointer. It must not grow every time the doc is regenerated — `/change-mode` updates the computed values **in place** and never appends new narrative lines.
- Each artifact gets only:
  - **Version number in the title line.** Example: `# Nestfully AI — User Stories Breakdown (v0.6.1)`
  - **A one-line pointer to the centralized changelog.** Example: `**Change history:** see [../changelog/[feature]-changelog.md](../changelog/[feature]-changelog.md)`
- All per-version history lives in `changelog/[feature]-changelog.md` — a single feature-level file **organized into one `## ` section per artifact** (PRD, System Design, User Stories Breakdown, Design Catalog, Timeline, …), each section append-only and dated — plus per-run `change-[date]-summary.md` files. `_pipeline-state.json` → `change_history[]` is the structured mirror.

### No inline decision log

- **The decision log is a sidecar, not a PRD section.** The running log of locked product/scope decisions lives in `decisions/[feature]-decision-log.md`, not inside the PRD body. The PRD's "Decision Log" section is reduced to a one-line pointer: `**Decision Log:** see [../decisions/[feature]-decision-log.md](../decisions/[feature]-decision-log.md)`.
- The decision log file is its own stakeholder-readable document (Topic / Chosen option / Rationale / Date per entry). Every step that previously wrote "to the PRD decision log" now appends to this file instead. Every step that reads the decision log (gate checks, exec summary, validation, self-checks) reads it from this file.

### What this rule does NOT touch (stays as content in artifacts)

- **Product-version concepts inside content** (e.g., "v1 ships in Oct, v1.x absorbs REA, v2 retires it") — that's roadmap, not file-version churn.
- **"Source PRD" / "consumes" pointer lines** — metadata about which versions of *other* artifacts this file was generated against.

This applies to all generated artifacts: PRD, system design, visual diagram, design catalog, design prompts, user-stories breakdown, timeline (markdown + HTML), sync reports, executive summary, and any future artifact types.
