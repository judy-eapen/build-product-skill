# Share for Review

Post a pipeline artifact to a Slack channel with a Confluence link and tagged reviewers.
Composes a structured review request and posts it via Slack MCP if connected, or outputs
it formatted for manual paste if not.

Pair with `/read-feedback` to close the loop: once reviewers comment on the Confluence
page, `/read-feedback` pulls those comments back and suggests PRD updates.

---

## Step 0 — Detect Slack MCP

Before asking the PM anything, silently check whether a Slack MCP is available.

Try calling the Slack MCP's channel list tool. Common tool names to try in order:
- `slack_list_channels`
- `mcp__slack__list_channels`
- `slack_conversations_list`

If any of these succeeds: set `slack_available = true`. Note the tool naming convention
that worked — use it for all subsequent Slack calls in this session.

If all fail or return a connection error: set `slack_available = false`. Do not mention
this to the PM yet — wait until after composing the message.

---

## Step 1 — What to share

Read `_pipeline-state.json` → `confluence_hub.artifacts` to see what's already published.

Ask the PM:

```
What do you want to share for review?

1. The whole feature hub (recommended — reviewers see the full picture and can comment on any page)
   → [Parent hub URL from confluence_hub.parent_page_url]

2. One specific artifact:
   [list each published child page from confluence_hub.artifacts, e.g.:]
     a. Step 3: PRD                         — [child URL]
     b. Step 8: Design Catalog — Phase 1    — [child URL]
     c. Step 10: User Stories Breakdown     — [child URL]
     d. Step 10½: Timeline                  — [child URL]

3. Multiple artifacts (one Slack post per artifact)

Which? (1, 2[letter], 3, or describe)
```

If the PM chose option 1 (whole hub), use the parent hub URL for the share. The Slack message will include just one link.

If the PM chose option 2 (one artifact), use that child page URL.

If the PM chose option 3 (multiple), repeat the share request flow once per selected artifact.

### If no hub exists yet

If `confluence_hub.parent_page_id` is null (Confluence has never been published for this feature):

```
No Confluence pages found for this feature. Options:
1. Publish everything to Confluence now, then share the hub URL (recommended)
2. Share the local file paths as plain text (less useful for reviewers — they can't comment)
3. Cancel

Which? (1 / 2 / 3)
```

If the PM chooses to publish: read and follow `subprompts/publish-to-confluence.md`, then return here with the resulting hub URL.

### Legacy single-PRD page

If state has only `export_urls.confluence_page` (pre-v2.3.0 feature), treat that URL as the PRD page and offer to share it. Also suggest the PM run `/publish-to-confluence` to migrate to the hub model so reviewers can see research, reviews, designs, etc.

---

## Step 2 — Collect review details

Ask in one block (not one question at a time):

```
A few quick details for the review request:

1. Slack channel: (e.g. #product-reviews, #eng-leads, or paste a channel ID)
2. Tag for review: (names, @handles, or roles — e.g. "@sarah, @tech-lead, @design-lead")
3. Review deadline: (e.g. "Friday EOD", "by next Tuesday", "48 hours")
4. What kind of feedback? (choose one or describe your own)
   a. Open feedback — any thoughts welcome
   b. Approve or comment — looking for a go / no-go
   c. Specific questions — I have specific things I want input on
5. (Optional) Anything specific you want reviewers to focus on?
```

Wait for answers. If the PM skips any field, use these defaults:
- Channel: ask again — no default, channel is required
- Deadline: "no hard deadline — sooner is better"
- Feedback type: open feedback
- Focus area: none

---

## Step 3 — Compose the Slack message

Build the message using this template. Fill in from the PRD executive summary and the
review details collected in Step 2.

```
👋 Review request — [Feature Name]

*What:* [1-sentence description from PRD executive summary]
*Why now:* [Gate just passed / milestone reached — e.g. "PRD approved, about to start design"]
*Asking for:* [feedback type from Step 2]
*Deadline:* [deadline]

[For each artifact being shared:]
📄 *[Artifact name]:* [Confluence URL]

[If specific focus area:]
🎯 *Focus on:* [PM's note]

[Tagged reviewers] — would love your input.
```

Show the composed message to the PM:

```
━━━ Review request message ━━━

[composed message]

━━━

Edit anything above, or say "looks good" to send.
```

Wait for PM confirmation or edits before proceeding.

---

## Step 4 — Post or output

### If `slack_available = true`:

Resolve channel and user IDs:
1. If the PM gave a channel name (e.g. `#product-reviews`): call `slack_list_channels`
   and find the matching channel ID.
2. If the PM gave @handles: call `slack_get_users` (or `slack_users_list`) and resolve
   each handle to a Slack user ID for proper `<@USERID>` mentions.
3. Reformat the message replacing @handle with `<@USERID>` for proper mentions.

Post the message:
- Call `slack_post_message` with `channel` = channel ID and `text` = formatted message.
- If post succeeds: record the message timestamp and channel in `_pipeline-state.json`
  under `review_requests[]`:
  ```json
  {
    "artifact": "[artifact name]",
    "confluence_url": "[url]",
    "slack_channel": "[channel name]",
    "slack_ts": "[message timestamp]",
    "deadline": "[deadline string]",
    "reviewers": ["[name1]", "[name2]"],
    "posted_at": "[ISO timestamp]",
    "feedback_read": false
  }
  ```
- Print:
  ```
  ✓ Posted to [#channel] — [Confluence URL]
  Tagged: [reviewer names]
  Deadline: [deadline]

  When reviewers have commented, run /read-feedback to pull their input back.
  ```

### If `slack_available = false`:

```
━━━ Slack MCP not connected ━━━

No Slack MCP is available in this session. Here is the formatted message to paste
manually into Slack:

─────────────────────────────────────────
[composed message — plain text, no markdown formatting that won't render]
─────────────────────────────────────────

Copy and paste this into [#channel name the PM gave].

When you have a Slack MCP connected, this command will post automatically.
When reviewers have commented on Confluence, run /read-feedback to pull their input back.
━━━
```

Also write the review request to `_pipeline-state.json` under `review_requests[]` as
above, with `slack_ts: null`.

---

## When to run

- After Gate 1 (PRD approval): share PRD for async stakeholder review before design begins.
- After Gate 2 (design approval): share design catalog for engineering lead review.
- After Gate 3 (Work pipeline): share user stories breakdown for engineering estimate review.
- Standalone: `/share-for-review` at any point with any artifact.

## Important

- Never post without PM confirmation of the composed message (Step 3).
- Never guess channel names. If the PM gives an ambiguous channel name and `slack_list_channels`
  returns multiple matches, list them and ask which one.
- If tagging fails for a specific user (user not found in Slack workspace), flag it:
  "Could not find [name] in the Slack workspace — will include as plain text instead."
  Do not block the post for one unresolved mention.
