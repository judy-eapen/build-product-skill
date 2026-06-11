# Export Conversation Transcript

Runs as **Step 12** at the very end of the pipeline (after Step 11 Export), OR can be called standalone via `/export-transcript` any time.

Writes the conversation between the PM and the model — every message back-and-forth that built this feature — to two markdown files in the feature's workspace. The PM gets both a clean reading version and a full forensic version of the exact same session.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 0 — Input Check

**If running inside the orchestrator (called from `/build-product` Step 12):** feature name is in conversation context. Proceed.

**If running standalone (called via `/export-transcript`):** ask the PM:

> "Which feature's conversation should I export?
> 1. **The most recently active feature** — derived from the most recently modified `_pipeline-state.json` under `~/Desktop/Resources/PDLC Workflow Docs/`.
> 2. **A specific feature** — give me the feature name (matches a subfolder name).
>
> (Default: 1)"

Apply the standard feature-name derivation rule (spaces → hyphens, lowercase) if the PM types the feature name freely.

---

## Step 1 — Locate the current session's transcript file

Claude Code persists every session as a JSONL file under `~/.claude/projects/`. The subfolder name is derived from the user's home directory path (slashes replaced with hyphens — e.g., `/Users/alice` → `-Users-alice`, `/home/bob` → `-home-bob`). To discover the correct folder at runtime:

```bash
ls ~/.claude/projects/
```

Each subfolder contains `[session-uuid].jsonl` files (one per session).

To find the current session's transcript:

```bash
ls -t ~/.claude/projects/*/*.jsonl | head -1
```

This returns the most-recently-modified `.jsonl` — which is always the current live session, because the file is appended to on every message.

Capture this path. Read it back to the PM with the file's last-modified timestamp so they can confirm it's the right session:

```
Found session transcript: [path]
Last modified: [timestamp]
Lines: [N]

Use this transcript? (yes / pick a different one)
```

If the PM says "different one", list the 5 most-recently-modified `.jsonl` files with their first/last timestamps so they can pick.

---

## Step 2 — Parse the JSONL

Each line in the JSONL is one event. The line `type` field determines what it is. Relevant types:

| type | What it is | Include in clean? | Include in full? |
|---|---|---|---|
| `user` | User message — the PM typing | ✅ yes | ✅ yes |
| `assistant` | Model response | ✅ yes (text blocks only) | ✅ yes (all blocks) |
| `system` | System reminder / hook output | ❌ no | ✅ yes |
| `attachment` | File / image attached to a user message | ✅ note attachment name | ✅ full metadata |
| `permission-mode` | Permission mode change | ❌ no | ✅ yes |
| `file-history-snapshot` | Snapshot of edited files | ❌ no | ❌ no (noise) |
| `ai-title` | Internal title-generation event | ❌ no | ❌ no |
| `last-prompt` | Bookmark of last prompt | ❌ no | ❌ no |
| `queue-operation` | Queue state event | ❌ no | ❌ no |

For `user` and `assistant` entries:
- `message.role` is `"user"` or `"assistant"`.
- `message.content` is either a string (user messages typed as plain text) or a list of content blocks.
- For lists, each block has a `type` field:
  - `text` — visible text. Include in both clean and full.
  - `thinking` — model's internal reasoning. **Exclude from both clean and full** (this is meant to be hidden from the user).
  - `tool_use` — a tool call. Exclude from clean; include in full with tool name + input.
  - `tool_result` — a tool result. Exclude from clean; include in full but **truncate to first 10 + last 5 lines** if the result is longer than 30 lines (tool results can be huge file dumps).

Each entry also has a `timestamp` (ISO-8601). Preserve it.

---

## Step 3 — Filter to this feature's pipeline run

The transcript file may span multiple features and conversations. Narrow to just the relevant portion.

**Lower bound (when to start):**
- Read `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/_pipeline-state.json` and use its **earliest** `pipeline_started_at` (or, if absent, the file's creation time).
- Filter to JSONL entries whose `timestamp >= pipeline_started_at`.

**Upper bound (when to stop):**
- For the auto-step (end of pipeline), the upper bound is **now** (include everything up to and including this conversation turn).
- For the standalone command, ask: "Up to now, or up to a specific date?" Default to now.

If `_pipeline-state.json` is missing (running on a feature that hasn't gone through `/build-product` yet), ask the PM for a start timestamp explicitly, or default to "the entire transcript file".

Note in the output the timestamp window: `Transcript window: [start] → [end]`.

---

## Step 4 — Compose the clean version

Filename: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/transcript/[feature-name]-transcript-clean.md`

Before composing, generate (or read cached) timing data: read `ai-framework/09-pipeline-timing.md` and produce the report. Cache the totals from `_pipeline-state.json` → `timing_report` to embed at the top of the transcript.

Format:

```markdown
# Conversation Transcript (Clean) — [Feature Name]

**Pipeline started:** [pipeline_started_at]
**Pipeline completed:** [pipeline_completed_at or "still running"]
**Transcript window:** [start] → [end]
**Source:** `~/.claude/projects/[your-profile]/[session-uuid].jsonl`
**Exported:** [now]

## Pipeline timing

| Metric | Time |
|---|---|
| **Wall-clock** | [Xh Ym Zs] |
| **Active work** | [Xh Ym Zs] |
| **Gate-wait total** | [Xh Ym Zs] |

See `timing/[feature]-timing.md` for the per-step breakdown.

---

This file contains every back-and-forth between the PM and the model for this feature's pipeline run. Tool calls, file reads, and the model's internal reasoning are excluded — see the `-full.md` companion file for those.

---

## [YYYY-MM-DD HH:MM] — You

[user message text, verbatim, as the PM typed it]

## [YYYY-MM-DD HH:MM] — Model

[assistant text-block content, concatenated if multiple text blocks in one message]

## [YYYY-MM-DD HH:MM] — You

[next user message]

...
```

Rules for the clean version:
- One section per message (alternating "You" / "Model").
- Drop any assistant message that contains **only** `thinking` and `tool_use` blocks (no text). The PM never saw model output for those turns.
- If a user message includes an attachment (image, file), note it: `[Attached: filename.png]` on its own line before the user text.
- Preserve markdown formatting the model produced (headers, lists, tables, code blocks).
- Do **not** include system reminders, permission-mode events, or hook outputs.

---

## Step 5 — Compose the full version

Filename: `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/transcript/[feature-name]-transcript-full.md`

Same header as the clean version, but the body includes everything visible in the conversation pane plus the tool activity the PM didn't see directly:

```markdown
## [YYYY-MM-DD HH:MM] — You

[user message]

## [YYYY-MM-DD HH:MM] — Model

[text blocks]

### Tool calls this turn

- **`Read`** — `/path/to/file.md`
- **`Bash`** — `git status` — *description: "Show working tree status"*
- **`Edit`** — `/path/to/file.md` (1 change)

### Tool results (abbreviated)

<details>
<summary>Read /path/to/file.md (full output collapsed)</summary>

```
[first 10 lines]
...
[last 5 lines, plus a "(N lines total)" footer]
```

</details>

## [YYYY-MM-DD HH:MM] — System reminder

[content of system message — useful for forensic debugging of harness behavior]

...
```

Rules for the full version:
- Always include `tool_use` and `tool_result` blocks, but **truncate any single result to first 10 + last 5 lines** if it's longer than 30 lines. Show `(N lines total — truncated)` after.
- Tool result text containing the full content of an artifact (PRD, user-stories doc, etc.) should be truncated aggressively — the artifact itself lives on disk; the transcript only needs to record that it was read.
- Use `<details>` / `<summary>` so the full version stays readable when opened in a markdown viewer (long tool outputs collapse by default).
- **Still exclude `thinking` blocks.** They are not part of the conversation record — they are the model's private reasoning.
- Include system reminders and permission-mode changes (these are useful when the PM is trying to figure out why a particular step behaved a certain way).

---

## Step 6 — Write files

Confirm the output paths to the PM before writing:

```
Writing:
  ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/transcript/[feature-name]-transcript-clean.md   ([N] messages, ~[N]KB)
  ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/transcript/[feature-name]-transcript-full.md    ([N] messages + [N] tool calls, ~[N]KB)
```

Create the `transcript/` subfolder if it doesn't exist. Write both files.

If a prior transcript export exists at the same paths, **append a timestamp suffix to the older files** before writing the new ones:

```
[feature-name]-transcript-clean.md         ← new (latest)
[feature-name]-transcript-clean.2026-05-21T16-30Z.md   ← prior (renamed)
```

That way the PM keeps a history of every export run.

Update `_pipeline-state.json` → `transcript`:

```json
"transcript": {
  "last_exported_at": "ISO-8601",
  "session_id": "...",
  "source_jsonl": "/path/to/session.jsonl",
  "clean_path": "...transcript/[feature]-transcript-clean.md",
  "full_path": "...transcript/[feature]-transcript-full.md",
  "window_start": "ISO-8601",
  "window_end": "ISO-8601",
  "message_count_user": 0,
  "message_count_assistant": 0,
  "tool_call_count": 0
}
```

---

## Step 7 — End-of-run report

```
━━━ Transcript exported — [Feature Name] ━━━

📖 Clean reading version:  [path]
🔍 Full forensic version:  [path]

Captured:
  [N] user messages
  [N] assistant replies
  [N] tool calls (full version only)
  Window: [start] → [end]
  Session: [session-uuid]

Open the clean version to re-read the back-and-forth. The full version is for digging
into what tools the model used, what it read, and what it edited at any given step.
━━━
```

If this is the final step of a `/build-product` run, print the pipeline-complete banner immediately after.

---

## Rules

- **Source is the JSONL on disk.** Never reconstruct messages from memory or context — read the file. This is the only way to capture the entire run, including parts that scrolled out of the visible context window.
- **Pick the right session.** Always confirm with the PM if there's any ambiguity about which session to export.
- **Exclude `thinking` blocks unconditionally.** They are the model's private reasoning. Not part of the conversation.
- **Truncate large tool results.** A 5000-line file read in the transcript helps no one — the file itself is on disk if anyone needs it.
- **Never delete prior transcripts.** Rename them with a timestamp suffix before writing new ones.
- **No external upload, no third-party API.** This step writes local files only. The transcript may contain anything from the conversation — names, internal links, decisions — and must not leave the user's machine.
- **Update `_pipeline-state.json`** at the end so the next run knows the prior export window.
