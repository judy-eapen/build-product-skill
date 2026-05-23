v2.12.0 — 2026-05-23

/push-to-figma command added — MCP-driven Figma frame generation from design prompts.

**New: `/push-to-figma` standalone command.** Generates real, editable Figma frames programmatically from a feature's design prompts file via the Figma MCP. Each frame is wired to the team's design system color variables (and components where they fit) — when brand tokens change, every frame updates. Output is the standard design catalog at `design/[feature]-figma-catalog.md` with one row per frame and its direct node URL. Companion to `/design-prompts`: the typical flow is `/design-prompts` → review → `/push-to-figma`. No v0 equivalent (v0 has no programmatic-push API).

Validated on the nestfully-ai feature — 29 mobile frames across 13 categories, generated against the 🐦 Nestfully Mobile library.

---

v2.11.0 — 2026-05-22

/exec-summary skill added + artifact-convention rule (no inline changelogs).

**New: `/exec-summary` standalone command.** Synthesizes a feature's PRD, system design, user-stories breakdown, timeline, and decision log into a single ~20 KB / ~5–7 PDF page executive summary structured for exec readability: vision, use cases, moving pieces, teams, timeline, open decisions, risks, the ask, what's next. Outputs markdown + PDF + DOCX side-by-side in the feature's root folder. Idempotent — re-running overwrites the same three files.

Standalone only — not auto-invoked by `/build-product`. PMs run it whenever an exec or stakeholder needs the "what is this and why" without wading through a 170 KB PRD. PRD is required; other inputs are opportunistic — missing artifacts produce a thinner section with a bracketed note rather than failing.

**New rule: no inline version-history blocks in generated artifacts.** Added to `ai-framework/style-preferences.md` (new "Artifact Conventions" section) and reinforced in `05-change-propagation.md` (Step 4 Writing). Going forward, artifacts get version in the title line + a single changelog pointer line; no `**v0.X (date):** ...` bullets, no "## Change log" sections inside the file. Full history lives in `changelog/[feature]-changelog.md` + per-run summary files + `_pipeline-state.json` → `change_history[]`. `/change-mode` and the generative commands all respect this.

See CHANGELOG.md for full version history.
