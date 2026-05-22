v2.5.0 — 2026-05-21

The HTML Gantt is now editable in the browser. Drag bars to shift start dates; drag the right edge to resize duration; downstream bars auto-cascade (hold Shift to lock). Edits auto-save to localStorage. Click Export Plan to download a JSON; run `/timeline apply [path]` to round-trip the edits back into the skill — updates `_pipeline-state.json`, re-renders the markdown sidecar with the new dates, and re-renders the HTML so the applied edits become the new baseline. New `timeline.applied_edits[]` history in state. Vanilla JS, no CDN, still opens offline.

See CHANGELOG.md for full version history.
