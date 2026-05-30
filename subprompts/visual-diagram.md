# Visual Diagram

Generate a standalone visual diagram for an existing PRD or a described feature. Use when:
- You have an approved PRD and want a diagram for a kickoff meeting or async share-out.
- You want a quick visual of a feature before writing the PRD.
- You want to update / regenerate the diagram for a feature without running the rest of the pipeline.

Read and follow `ai-framework/03c-visual-diagram.md`.

## Inputs

The underlying prompt collects inputs as Step 0:

- **Required:** a PRD or a feature description. The skill will ask you to paste the PRD, give a file path, or describe the feature in your own words.
- Feature name (asked if not in conversation context).

No prior pipeline run is required. If you only have a description (no PRD), the diagram is still produced but the output will note: "Generated from PM description only — no PRD traceability available."

## Output

`~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md`

Includes:
- The diagram **in Figma / FigJam** (created via the Figma MCP) — the output file carries the `[Open in Figma](URL)` link. If the Figma MCP is unavailable, a clearly-labeled **temporary Mermaid fallback** is produced instead, flagged for regeneration once Figma is connected.
- A traceability table mapping each node back to its source PRD user story (when a PRD is provided).

## Diagram types

The skill auto-suggests based on the PRD content but you can override:

- **User journey map** — touchpoints, decisions, emotions. Best for consumer-facing features.
- **System architecture diagram** — components, services, APIs, data stores. Best for backend-heavy features.
- **Wireflow** — wireframe-level UI sketches with flow arrows. Best for UI-heavy features with multiple screens.

## When to use the full pipeline instead

If you want the diagram to inform downstream design and Jira ticket creation, run `/build-product` and pick the Work pipeline. The diagram there flows into Step 8 (Design) and Step 10 (User Stories Breakdown) automatically.
