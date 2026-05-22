v2.6.0 — 2026-05-21

Two refinements to the v2.5.0 editable Gantt that drop the round-trip from three steps to two:

1. New **💾 Save to skill** button in the HTML Gantt (Chrome / Edge). Uses the File System Access API to write the plan JSON directly into the feature's `timeline/` folder — no Downloads round-trip. File handle persists across sessions via IndexedDB so subsequent saves are silent. Safari/Firefox fall back to download.

2. **Auto-discovery on `/timeline apply`**. The command now accepts zero arguments and scans the feature's `timeline/` folder and `~/Downloads/` for the latest plan JSON.

Typical flow now: edit in browser → click 💾 Save → type `/timeline apply` in chat. No paths to type.

See CHANGELOG.md for full version history.
