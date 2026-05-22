# Export Conversation Transcript

Write the full back-and-forth between the PM and the model for a feature's pipeline run to two markdown files — a clean reading version and a full forensic version.

Use when:
- The pipeline just finished and you want a record of every message you sent and every reply you got, in one document.
- You're debugging why a particular step turned out the way it did and need to re-read the conversation that led to it.
- You're handing the feature to a teammate and want them to see the conversational context, not just the artifacts.

Read and follow `ai-framework/08-export-transcript.md`.

## Inputs

The underlying prompt collects inputs as Step 0:

- **Required:** feature name (asked if not in conversation context).
- **Auto-detected:** the current session's transcript file (Claude Code persists every session as a JSONL under `~/.claude/projects/-Users-judydarvin/`; the prompt picks the most-recently-modified file and confirms with the PM).
- **Optional:** time window. Defaults to "from when this feature's pipeline started" (read from `_pipeline-state.json` → `pipeline_started_at`) through "now".

No prior pipeline run is strictly required, but the time-window auto-detection works best when one is available.

## Output

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/transcript/
├── [feature-name]-transcript-clean.md   ← user + assistant text only
└── [feature-name]-transcript-full.md    ← everything: text + tool calls + system events
```

Both files cover the same conversation window. The clean version is what you'd want to re-read; the full version is what you'd want for forensic debugging.

If prior transcript exports exist at the same paths, they are renamed with a timestamp suffix (e.g. `[feature]-transcript-clean.2026-05-21T16-30Z.md`) before the new files are written — so you keep a history of every export, never overwriting prior runs.

The model's `thinking` blocks are excluded from both files (they are private reasoning, never part of the visible conversation). Long tool results are truncated to first 10 + last 5 lines in the full version, with a "(N lines total)" footer.

## When to run

- **Automatically at the end of `/build-product`** — runs as Step 12, after the parallel Step 11 Export. Can be skipped for a single run by saying "skip transcript" at the end of Step 11.
- **Standalone via `/export-transcript`** — re-export any time, even months later, to get a fresh markdown copy of the conversation.

## What it does NOT include

- The model's internal reasoning (`thinking` blocks). Never part of the conversation.
- File-history snapshots, queue events, or other internal Claude Code state events.
- Anything outside the time window (other features' conversations, prior unrelated sessions).
