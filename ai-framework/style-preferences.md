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

- **No inline version-history blocks in artifacts.** Do not embed `**v0.X (date):** ...` bullets or "what changed in this version" callouts at the top (or anywhere) of an artifact file.
- Each artifact gets only:
  - **Version number in the title line.** Example: `# Nestfully AI — User Stories Breakdown (v0.6.1)`
  - **A one-line pointer to the centralized changelog.** Example: `**Change history:** see [../changelog/[feature]-changelog.md](../changelog/[feature]-changelog.md)`
- All per-version history lives in `changelog/[feature]-changelog.md` (append-only) plus per-run `change-[date]-summary.md` files. `_pipeline-state.json` → `change_history[]` is the structured mirror.
- This applies to all generated artifacts: PRD, system design, visual diagram, design catalog, design prompts, user-stories breakdown, timeline (markdown + HTML), sync reports, executive summary, REA work brief, and any future artifact types.
- **Distinct from this rule** — these are content, not file-version metadata, and stay in artifacts:
  - PRD Decision Logs (#45, #46, ...) — they record product/scope decisions.
  - Product-version concepts inside content (e.g., "v1 ships in Oct, v1.x absorbs REA, v2 retires it").
  - "Generated" / "Source PRD" pointer lines — they're metadata about which versions of other artifacts this file consumes.
