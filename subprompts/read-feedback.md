# Read Feedback

Pull reviewer comments from a Confluence page, synthesize them into actionable
suggestions, and apply approved changes to the PRD. Closes the async review loop
opened by `/share-for-review`.

---

## Step 0 — Identify the Confluence page

Check `_pipeline-state.json` → `review_requests[]` for any entries where
`feedback_read = false`.

If one or more unread review requests exist, present them:

```
Pending review requests:

1. PRD — [Confluence URL] — posted [date] — deadline [deadline]
   Reviewers: [names]
   [N days since posted]

2. Design catalog — [Confluence URL] — posted [date]
   ...

Which review are you reading feedback for? (number or "all")
```

If no review requests are in state (PM ran `/share-for-review` in a prior session or
manually), ask:

```
Which Confluence page should I read feedback from?
Paste the page URL or title.
```

---

## Step 1 — Pull comments from Confluence

For each selected Confluence page:

1. Call `getConfluencePageInlineComments` to get inline comments (anchored to specific
   paragraphs or text selections in the page body).
2. Call `getConfluencePageFooterComments` to get footer/general comments.
3. For each comment, extract:
   - Author name
   - Comment text
   - Anchor text (for inline) — the paragraph or sentence the comment is attached to
   - Created date
   - Resolved status (skip already-resolved comments)

If no comments are found:

```
No unresolved comments on [page title] yet.

Options:
1. Check again later — run /read-feedback when reviewers have commented.
2. Send a reminder via Slack — I can compose a follow-up message to the channel.

Which? (1 / 2 / skip)
```

If PM chooses reminder: compose a brief Slack follow-up ("Just a reminder — feedback
on [feature name] is due [deadline]. Link: [URL]") and post or output it per the same
Slack MCP detection logic used in `/share-for-review`.

---

## Step 2 — Attribute and group comments

Group comments by the PRD section they relate to. Use the anchor text (for inline
comments) to identify the relevant section. For footer comments with no clear anchor,
group under "General."

Build a comment map:

```
Section 1 — Executive Summary
  @sarah (2 days ago): "Success metric feels vague — what's the current baseline?"
  @tech-lead (1 day ago): "Agreed on the metric. Also — is 10% conversion realistic given current funnel?"

Section 4 — Data Model
  @tech-lead (1 day ago): "We don't have an index on user_id + created_at for this table.
   Queries will be slow at scale."

General
  @design-lead (3 days ago): "Looks good overall. One question on Section 7 phasing —
   why is Phase 2 before Phase 1b?"
```

---

## Step 3 — Synthesize into three categories

Read the full comment map and classify each comment:

**Category A — Suggested PRD edits (clear change requests)**
The reviewer is asking for a specific change to the PRD content. Include:
- What section is affected
- What the reviewer said
- What the suggested PRD edit would be (drafted, ready for PM approval)

**Category B — Open questions (PM decision required)**
The reviewer asked a question that requires the PM to make a judgment call before the
PRD can be updated. Not a direct edit request — an input gap.

**Category C — Agreements / positive signals**
Reviewer confirms they are happy with a section or decision. No action needed.
Record these — they are evidence that a decision is validated.

---

## Step 4 — Present synthesis to PM

```
━━━ Feedback synthesis — [Feature Name] ━━━

[N] comments from [N] reviewers across [N] sections.

─── SUGGESTED EDITS ([N]) ───

[1] Section 1 — Success metric
    @sarah + @tech-lead both flagged this.
    Comment: "Success metric feels vague — what's the current baseline?"

    Suggested PRD edit:
    Current text: "Increase user activation"
    Proposed:     "Increase user activation from 34% (current baseline, May 2026)
                   to 42% within 60 days of launch"

    Approve this edit? (yes / no / edit it)

[2] Section 4 — Data Model
    @tech-lead flagged this.
    Comment: "We don't have an index on user_id + created_at for this table."

    Suggested PRD edit:
    Add to the indexes field for the events table:
    "Compound index on (user_id, created_at DESC) — required for timeline queries"

    Approve this edit? (yes / no / edit it)

─── OPEN QUESTIONS ([N]) ───

[1] @design-lead: "Why is Phase 2 before Phase 1b?"
    → Needs PM decision before proceeding. Answer here or flag as Open Question in PRD.

    Your answer (or say "add to open questions"):

─── AGREEMENTS ([N]) ───

✓ @tech-lead confirmed the API contract in Section 5 looks correct.
✓ @sarah confirmed the target user definition in Section 1 is accurate.

These decisions are now reviewer-validated. Add to PRD decision log? (yes / skip)
━━━
```

Process approvals one at a time or accept "approve all" to approve all Category A edits.

For "edit it" responses: show the current text and let the PM provide the replacement before applying.

---

## Step 5 — Apply approved changes

For each approved edit:

1. Read the current PRD file at the path in `_pipeline-state.json`.
2. Apply the edit in place.
3. Add a decision log entry:
   ```
   [Date] — [Section] updated based on reviewer feedback from [reviewer name(s)].
   Change: [one-line summary of what changed]
   ```
4. For Category C agreements the PM approved adding to the decision log: add an entry
   marked `[Reviewer-validated — [name(s)], [date]]`.

For each open question the PM answered inline:
- Add the answer to the relevant PRD section or to Section 11 (Open Questions) with
  the resolution noted.

After all edits are applied:

```
✓ PRD updated — [N] edits applied, [N] open questions resolved.
  Decision log updated with [N] reviewer-validated decisions.
```

---

## Step 6 — Sync back to Confluence

Ask:

```
Re-sync the updated PRD to Confluence so reviewers see the final version?
(yes / no)
```

If yes: call `updateConfluencePage` with the updated PRD content. If the MCP is
unavailable, apply Error Type 4 from `ai-framework/error-handling.md`.

After syncing, resolve the addressed comments on the Confluence page where possible.
If `resolveConfluenceComment` (or equivalent) is available, resolve each comment
whose corresponding PRD edit was approved.

---

## Step 7 — Close the review loop

Update `_pipeline-state.json`:
- Set `feedback_read = true` for this review request.
- Add `feedback_applied_at` timestamp.
- Add `edits_applied: N` count.

Print final summary:

```
━━━ Review loop closed — [Feature Name] ━━━

Comments read:   [N]
Edits applied:   [N]
Questions resolved: [N]
Agreements logged: [N]

PRD: [path]
Confluence: [URL] (updated)

Next step:
- If Gate 1 was open, the PRD is now review-informed. You can proceed to Gate 2.
- If this was a post-Gate review, run /share-for-review again to notify reviewers
  that their feedback has been incorporated.
━━━
```

---

## Important

- Never apply an edit without PM approval for that specific edit. Never batch-apply
  without confirmation.
- If two reviewers left conflicting suggestions on the same section, surface as a
  conflict card (Error Type 2 from `ai-framework/error-handling.md`) and require PM
  resolution before applying either edit.
- Do not resolve Confluence comments that were marked "no" (rejected). Leave them
  visible so the PM can reply explaining why they were not incorporated.
- If the PRD has already passed a gate, check (Error Type 3) whether the edit
  contradicts any locked decision in the decision log. If it does, flag before applying.
