# Push Designs to Figma — MCP-Driven Frame Generation

Generates real Figma frames programmatically from the design prompts file via the Figma MCP. Creates one frame per screen, wired to your team's design system color variables (and components where they fit). Output is editable Figma frames, not images — they remain linked to the source design system, so when brand tokens change, the frames update automatically.

**Prerequisite:** Figma MCP must be connected (`claude.ai Figma`). If it isn't, ask the PM to install it before proceeding.

**Companion to `/design-prompts`.** `/design-prompts` generates the text prompts; this command turns them into Figma frames. Either can run standalone, but the typical flow is `/design-prompts` → review → `/push-to-figma`.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 0 — Input Check

Before doing anything, confirm:

1. **Prompts file exists** at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-design-prompts.md`. If not present, ask the PM for a path or suggest running `/design-prompts` first.
2. **Figma MCP is connected.** A quick check: try `whoami` on the Figma MCP. If it fails, ask the PM to connect the MCP and try again.
3. **Source design-system Figma URL.** Check `_pipeline-state.json` → `intake.figma_design_system_url`. If present, confirm with the PM. If not, ask for the URL of the Figma file that contains the team's brand colors, type, and components.
4. **Product type from intake.** Read `intake.product_type` to set frame dimensions:
   - Web app → 1440 × 900 (desktop)
   - Mobile app → 390 × 844 (iPhone 14 Pro)
   - Other → ask the PM

Ask the PM:
1. **Scope:** all categories from the prompts file, or a subset (e.g., "just Phase 1 screens" / "just B + G")?
2. **Target file:** create a new Figma file (recommended) or push to an existing file? If existing, ask for the file URL.
3. **Fidelity:** **production-trace** (one frame per screen state with full content) or **summary** (one representative frame per category, ~5–8 frames total). Summary is faster; production-trace covers every state in the prompts file. Default: production-trace.

Wait for answers.

---

## Step 1 — Load Required Skills + Discover Design System

**MANDATORY before any `use_figma` call** — load the Figma MCP skills:

1. Read the `figma-use` skill from `skill://figma/figma-use/SKILL.md` (mandatory; covers font loading, color ranges, page rules, layout sizing).
2. Read the `figma-generate-design` skill from `skill://figma/figma-generate-design/SKILL.md` (mandatory for multi-section screen builds).

Both skills are loaded via MCP resources, so pass `resource:figma-use,resource:figma-generate-design` as the `skillNames` parameter on every `use_figma` call.

Then discover the design system. Run these in parallel:

1. `get_libraries({ fileKey: <source design system file> })` — find all subscribed libraries. Identify the team's primary library by name.
2. `whoami()` — get the user's plans (need `planKey` if creating a new file).
3. `search_design_system` queries to find:
   - Color variables: query "color", "brand", "primary", "neutral", "background", "surface". Capture variable keys + variable IDs.
   - Typography: query "heading", "body", "text style".
   - Key components: query "button", "input", "card", "top bar", "search", "list", "chip".

Save the discovered tokens to `_pipeline-state.json`:

```json
{
  "figma_generation": {
    "source_library_file_key": "<fileKey>",
    "source_library_key": "lk-<libraryKey>",
    "variables": {
      "primary_action": { "key": "<variableKey>", "name": "Branding Colors/Orange Medium 700" },
      "headline_text": { ... },
      "surface_tint": { ... },
      "secondary_text": { ... },
      "border": { ... },
      "alert_bg": { ... }
    },
    "components": { ... },
    "frame_dimensions": { "width": 390, "height": 844 },
    "target_file_url": "<set in Step 2>"
  }
}
```

Confirm to the PM: "Found N color variables and M components in [Library Name]. Proceeding to create the target file."

---

## Step 2 — Create or Resolve Target File

**If new file:**
1. From `whoami()` plans, ask the PM which team/org to create the file in if there's more than one. If there's one, use it.
2. Call `create_new_file({ fileName: "[feature-name] — v1 Screens (generated from PRD)", planKey, editorType: "design" })`.
3. Capture the returned `file_key` and `file_url`. Save to state under `figma_generation.target_file_url`.

**If existing file:**
1. Validate the URL — extract `file_key`.
2. Confirm with the PM: "I'll push frames into [filename]. Existing pages will not be deleted, but new pages (one per category) will be added."

---

## Step 3 — Read Prompts and Build Screen Inventory

Read the prompts file. Parse the category-level structure (A, B, C, …) and the individual frames described in each.

Build a screen inventory:

| Category | Frame # | Frame name | User story / source | Notes |
|---|---|---|---|---|
| A | 1 | Welcome | … | Phase 2 |
| A | 2 | Notifications permission | … | Phase 2 |
| … | … | … | … | … |

Present the inventory to the PM. Ask: "I'll generate N frames across M categories. Approved?"

Adjust if the PM removes / re-scopes items.

---

## Step 4 — Set Up Page Structure

In a single `use_figma` call:

1. Rename the default `Page 1` to `Cover`.
2. Create one page per category in alphabetical order: `A. [Category short name]`, `B. [Category short name]`, …
3. Return all page IDs as a structured object.

Save the page IDs to state under `figma_generation.page_ids`.

---

## Step 5 — Generate Frames (Per Category)

For each category, generate the frames it contains. **Follow the `figma-use` and `figma-generate-design` skill rules strictly:**

### Per-frame loop

For each frame in the category:

1. **Switch to the category page** at the start of the `use_figma` call: `await figma.setCurrentPageAsync(page)`. Call this **at most once per script.**
2. **Create the wrapper** — auto-layout VERTICAL at the target dimensions (e.g., 390 × 844). Position it horizontally next to existing frames on the page (every frame +430 px to the right of the previous one for mobile, +1500 for desktop).
3. **Build sections incrementally.** Per the skill: at most 10 logical operations per `use_figma` call. Common pattern: status bar + nav bar in call 1, content area in call 2, footer/composer in call 3.
4. **Reference design tokens via variable binding**, not hardcoded hex. Use `figma.variables.importVariableByKeyAsync(key)` to import each variable, then `figma.variables.setBoundVariableForPaint(...)` to bind it to a paint.
5. **Prefer real components when available.** Use `figma.importComponentSetByKeyAsync(key)` and create instances rather than building primitives. Fall back to primitives only when no component fits.
6. **Always `await` async calls** (font loads, page switch, variable/component imports). Batch in `Promise.all` for speed.
7. **Load fonts before mutating text.** Use the canonical recipe: `await figma.loadFontAsync({ family: "Inter", style: "..." })` → mutate `characters` → return IDs.
8. **Position new top-level nodes at known coordinates** — never trust the (0,0) default.
9. **Validate after each frame** with `get_screenshot({ nodeId: wrapperId })`. Check for: clipped text, overlapping elements, placeholder text not overridden, wrong variant chosen.

### Common frame structure (mobile)

| Section | Pattern |
|---|---|
| Status bar | HORIZONTAL auto-layout with time on left, signal icons on right |
| Top bar / nav bar | HORIZONTAL auto-layout with leading icon, centered title, trailing action |
| Content scroll area | VERTICAL auto-layout with `layoutSizingVertical = 'FILL'` (or `HUG` if a footer is pinned) |
| Footer / composer | VERTICAL auto-layout pinned to bottom with top border |
| Home indicator | Centered 134 × 5 rounded rectangle at the bottom |

### Common frame structure (web)

| Section | Pattern |
|---|---|
| Top nav | HORIZONTAL auto-layout, full width |
| Side nav (if applicable) | VERTICAL auto-layout, fixed width |
| Main content area | VERTICAL auto-layout, fills remaining space |
| Footer (optional) | HORIZONTAL auto-layout, full width |

### Error recovery

`use_figma` is atomic — a failed script makes no changes. On any error: stop, read the error message, fix, retry. **Common errors and fixes:**

- `FILL can only be set on children of auto-layout frames` → set sizing **after** appending to the auto-layout parent.
- `cannot read property 'remove' of undefined` → child you tried to remove doesn't exist (likely a conditional append). Guard with `if (parent.children.length > 0)` before `.remove()`.
- `not implemented` for `figma.notify()` → use `return` for output.
- Font style not found → call `figma.listAvailableFontsAsync()` and verify the family/style string before retry.

### Throughput

Expect **2–4 `use_figma` calls per frame** at production-trace fidelity. For a typical PRD (~30 frames), this is 60–120 use_figma calls total. Communicate progress to the PM after each category completes.

---

## Step 6 — Update Design Catalog

After all frames are generated, write the catalog to:

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-figma-catalog.md
```

Confirm the path before writing: `Writing: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-figma-catalog.md`

Catalog structure:

```markdown
# Design Catalog — [Feature Name] (Figma — generated via MCP)

**Tool used:** Figma (designs generated programmatically via Figma MCP)
**Phase:** [N or "All v1"]
**Date:** [YYYY-MM-DD]
**PRD:** [path]
**Prompts source:** [path]
**Design system referenced:** [Library Name] ([source design system URL])

**Generated file:** [target Figma file URL]

Brief description of the target dimensions and design tokens applied.

## [Category A name]

| Frame | URL |
|---|---|
| A.1 [Frame name] | [Direct node URL — file URL + ?node-id=X-Y] |
| A.2 [Frame name] | ... |

[Repeat per category]

## Design tokens applied

| Token | Variable key | Usage |
|---|---|---|
| [name] | [key] | [usage description] |

All bindings are live — if a variable changes in the source library, every frame here updates.

## Open design questions

[Carry forward from the prompts file; note any decisions taken during generation]

## Notes for the designer

- These are starting frames, not Gate-2-ready specs. Each frame has the right IA + right tokens; designer polishes typography hierarchy, spacing rhythm, illustrations.
- Status bar / nav bar / home indicator patterns repeat across screens — extracting as components would help iteration.
- [Specific notes per the generation run — e.g., known truncations, components not yet swapped in]

## Provenance

This catalog was generated programmatically from the prompts file via the Figma MCP `use_figma` API. The frames are real Figma frames (not images) and remain editable.
```

---

## Step 7 — Save State

Update `_pipeline-state.json` → `figma_generation`:

```json
{
  "figma_generation": {
    "target_file_url": "...",
    "target_file_key": "...",
    "page_ids": { "Cover": "0:1", "A. Onboarding": "1:2", ... },
    "frame_ids": {
      "A.1 Welcome": "9:10",
      "A.2 Notifications": "9:30",
      ...
    },
    "catalog_path": "~/Desktop/...",
    "generated_at": "YYYY-MM-DD HH:MM",
    "fidelity": "production-trace" | "summary",
    "frame_count": N
  }
}
```

This lets `/change-mode` and subsequent runs reference the existing file rather than recreating it.

---

## Step 8 — Report Back

Output to the PM:

```
✓ Pushed N frames to Figma → [target file URL]
Categories covered: [A, B, ...]
Design tokens applied: [count] from [library name]
Catalog written to: [path]

Open the file in Figma. Each page is one category. Hit "R" to zoom to fit.
```

---

## Rules

- **Mandatory skills.** Always load `figma-use` AND `figma-generate-design` from MCP resources before any `use_figma` call. Pass `resource:figma-use,resource:figma-generate-design` in the `skillNames` parameter on every call.
- **No hardcoded brand values.** Always reference team design tokens via variable binding. The whole point of this command (vs. v0) is that the output stays linked to the team's design system.
- **Production-trace ≠ Gate 2 spec.** Output is mid-fi: real IA, real tokens, illustrative copy. Typography hierarchy, illustrations, and per-state polish are the designer's job. The catalog should say this explicitly.
- **One page per category.** Keeps navigation simple in Figma. Sub-frames within a category land horizontally on the same page.
- **Frames are mobile by default for mobile-product types and desktop for web-product types.** Don't ask the PM unless the product type is "Other".
- **Idempotent only on file level, not on frames.** Re-running this command creates a *new* set of frames (likely on new pages) unless the PM explicitly requests overwrite. If overwrite is requested, fetch existing frames by name from state and modify in place — never delete and recreate, which would lose any designer edits.
- **Do not modify the source design system file.** Read-only on the source library. All writes go to the target file.
- **Per the figma-use skill** (10 ops per call), expect many small `use_figma` calls. This is by design — gives you per-call validation and atomic rollback.

---

## Limitations

- **No v0 equivalent.** v0 is a browser chat product with no public API for programmatic prompt submission. If the PM wants v0 output, run `/design-prompts` instead and paste manually.
- **Image fills not supported via this path.** `use_figma` cannot fetch external image URLs. Listing photos, hero images, etc. land as gray placeholder rectangles. If real images are needed, the designer drops them in manually post-generation (or run `generate_figma_design` in parallel for web apps with screenshots — see the `figma-generate-design` skill for that workflow).
- **Component instance text overrides** require the component's TEXT property keys. The discovery step in `figma-generate-design` covers this; budget extra time when the target library uses non-obvious component property keys.
