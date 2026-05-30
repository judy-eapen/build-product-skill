# Visual Diagram

Runs after Gate 1 and before the design step (when called from the orchestrator), OR can be called standalone via `/visual-diagram` when a PM has a PRD and just wants a diagram.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

**Figma / FigJam is the deliverable** (created via Figma MCP `generate_diagram`). The diagram lives in Figma — the tool the product team works in — can be shared with any team member, and embeds cleanly in Confluence. Figma is the single visible surface for the diagram; the diagram doc carries the Figma link, not a Mermaid block.

**Mermaid is a temporary fallback only — when the Figma MCP is unavailable.** It never ships alongside a successful Figma diagram. When the MCP isn't connected, produce a Mermaid block so the pipeline doesn't stall, but mark it clearly as temporary and prompt the PM to connect Figma so it can be replaced (see the fallback path in Step 2). Mermaid does not render in Confluence (or many markdown viewers) without a plugin — which is exactly why it is fallback-only.

---

## Step 0 — Input Check (gracefully handle standalone calls)

Before doing anything else, determine whether you have the PRD content available.

**If running inside the orchestrator (called from `/build-product`):** the PRD is already in conversation context at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`. Skip Step 0 and proceed to Step 1.

**If running standalone (called via `/visual-diagram` without a prior pipeline run):** ask the PM:

> "To build a diagram I need to understand the feature. Choose one:
> 1. **Paste the PRD content** here directly.
> 2. **Give me the file path** to an existing PRD (e.g., `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`).
> 3. **Describe the feature in your own words** if you don't have a PRD — I'll work from that, but the diagram won't have PRD traceability."

Wait for the PM's input. Do not proceed until you have something to work from.

Also ask for the feature name (used for the output file path). If running standalone with no PRD, derive the feature name from the PM's description and confirm with them. Apply the standard feature-name derivation rule (spaces → hyphens, lowercase).

If the PM only provided a description (no PRD), note explicitly in the output file: "Generated from PM description only — no PRD traceability available."

---

## Step 1 — Diagram Type Selection

Ask the PM which type they want. Present exactly three options:

1. **User journey map** — end-to-end user experience showing touchpoints, decisions, and emotions.
2. **System architecture diagram** — how components, services, APIs, and data stores connect.
3. **Wireflow** — wireframe-level UI sketches with flow arrows showing how screens connect.

If the PM is unsure, infer the best type from the PRD:
- User-facing consumer features → User journey map.
- API-heavy or backend-heavy features → System architecture diagram.
- UI-heavy features with multiple screens → Wireflow.

State the inferred choice and ask for confirmation before generating.

---

## Step 2 — Generate Diagram

### Primary path — Figma MCP

Load the `/figma-generate-diagram` skill before calling the Figma MCP. This skill is mandatory before calling `generate_diagram`.

Once the skill is loaded, call `generate_diagram` to create a FigJam diagram. The diagram must reflect the PRD exactly — every node must trace to a specific PRD user story or section. Do not add flows, screens, or components that are not in PRD scope.

After the diagram is created:
- Record the Figma diagram URL in `_pipeline-state.json` → `export_urls.figma_diagram_url`.
- Store the URL as a variable for use in Step 3 (output file) and downstream steps (Confluence embed, if publishing).

If `generate_diagram` fails or the Figma MCP is not available, fall back to the Mermaid path below.

### Fallback path — temporary Mermaid (only when Figma MCP is unavailable)

First tell the PM, plainly:

> "The Figma MCP isn't connected, so I can't create the diagram in Figma right now. I'll produce a temporary Mermaid version so the pipeline keeps moving — connect the Figma MCP and re-run `/visual-diagram` and I'll replace it with a real Figma diagram. Continue with the temporary version? (yes / I'll connect Figma first)"

If the PM connects Figma, return to the primary path. Otherwise generate the diagram in Mermaid syntax (every node must trace to a specific PRD user story or section), and:
- Set `export_urls.figma_diagram_url` to `null` in `_pipeline-state.json` so a later run (and `/publish-to-confluence`) knows the Figma diagram is still owed.
- Set `_pipeline-state.json` → `visual_diagram.needs_figma_regen = true`.
- Label the Mermaid block in the output file as a temporary fallback (see Step 3). Do not wrap it in HTML comments — keep it a visible fenced block so it renders where Mermaid is supported.

⚠ **Mermaid limitation:** Mermaid renders in Cursor / VS Code / some markdown viewers, but **not in Confluence** without a third-party plugin. This is the temporary cost of not having Figma connected — flag it so it gets replaced.

---

## Step 3 — Output

Confirm the file path to the PM before writing:

```
Writing: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md
```

Then write the diagram to that path. If the parent folders do not exist, create them automatically.

The output file must include:
- Diagram type chosen.
- Format used (Figma FigJam, or temporary Mermaid fallback).
- **If Figma (the normal case):** the `[Open in Figma](URL)` link as the diagram's single visible surface. Do not also paste a Mermaid block.
- **If temporary Mermaid fallback:** a bold banner at the top of the diagram section — `⚠ Temporary fallback — the Figma MCP was unavailable. Connect it and re-run /visual-diagram to replace this with a Figma diagram.` — followed by the visible Mermaid fenced block (not HTML-commented) and the Confluence limitation note.
- A traceability table mapping every node, screen, or component to its source PRD user story or section.

---

## Step 4 — Validation

Ask exactly one question:

"Does this diagram accurately represent the feature as scoped in the PRD?"

- If yes → proceed to the design step.
- If no → ask what is wrong and regenerate from Step 2. Do not skip the traceability check after regeneration.

---

## Rules

- Label every node, arrow, and state clearly.
- Use plain language labels unless the diagram type is system architecture (where technical terms are appropriate).
- Every screen or component in the diagram must trace back to a user story or PRD section.
- Do not add flows that imply scope not in the PRD. If the diagram exposes a missing flow, flag it as an open question to the PM rather than inventing it.
- Self-check before writing: does this diagram contradict any decision recorded in the decision log (`decisions/[feature]-decision-log.md`)? If yes, surface to the PM before writing.
- If using Figma FigJam, always load `/figma-generate-diagram` before calling `generate_diagram`.
