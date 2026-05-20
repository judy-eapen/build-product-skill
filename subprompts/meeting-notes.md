# Meeting Notes

Turn raw meeting notes into a clear, structured summary the PM can share and follow up on. User pastes or @mentions their notes; you output summary, decisions, action items, and open questions.

## When to use

- User has raw notes from a meeting (paste in chat or @mention a file).
- Works for any meeting type: standup, stakeholder review, kickoff, retro, 1:1, planning.
- Goal: make the notes usable—easy to skim, easy to turn into follow-up tasks.

---

## Your process

### 1. Get the notes

- Ask the user to paste their meeting notes or @mention a file (e.g. `@notes.txt` or a doc).
- If they haven’t provided context, ask: **Meeting name and date?** (e.g. "Q1 roadmap review, Feb 6")
- Optional: "Any attendees or key owners to call out?" (for action items)

### 2. Structure the output

Parse the raw notes and fill these sections. Use only what’s in the notes; don’t invent content.

| Section | What to include |
|---------|------------------|
| **Summary** | 2–4 sentences: what the meeting was about and the main takeaway. |
| **Decisions** | What was decided (with who/what/why if stated). Use bullets. |
| **Action items** | Who does what by when (if mentioned). Format: `[Owner] Action — [due/context]`. If no owner, use "TBD" or "Team". |
| **Open questions** | Unresolved questions, follow-up discussions, or "we need to decide…". |
| **Next steps** | Only if clearly stated (e.g. "Next meeting Tuesday to review"). Otherwise omit. |

- Keep the original wording where it’s clear; tighten and fix grammar only when it helps.
- If the notes are very short, still produce the structure and put "—" or "None noted" where there’s nothing.
- If the notes are in bullets/stream of consciousness, group related bullets into the right section.

### 3. Preserve important detail

- **Names and commitments:** If someone said "I’ll do X," that’s an action item with their name.
- **Dates and deadlines:** Keep them; add them to the relevant action item or decision.
- **Links or references:** Keep URLs, doc names, ticket IDs (e.g. Jira keys) in the right section.
- **Disagreements or risks:** Put in Open questions or a short "Risks / disagreements" bullet if it matters for follow-up.

### 4. Output format

Use this structure when you reply:

```markdown
# Meeting notes: [Meeting name]
**Date:** [Date if known]

## Summary
[2–4 sentences]

## Decisions
- [Decision 1]
- [Decision 2]

## Action items
- **[Owner]** [Action] — [due date or context]
- ...

## Open questions
- [Question 1]
- [Question 2]

## Next steps (optional)
- [e.g. Next meeting, next review]
```

### 5. Save (optional)

- Ask: "Save this to a doc?"
- If yes, write to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/meetings/[meeting-name]-[date].md` (e.g. `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/meetings/q1-roadmap-review-2025-02-06.md`). Use a short, readable filename.

---

## Important

- **Only use what’s in the notes.** Don’t add decisions or action items that weren’t stated or implied.
- **When in doubt, leave it in Open questions** (e.g. "Clarify: [thing that was unclear].").
- **Keep it short.** Summary and bullets should be scannable in under a minute.
- **Preserve ownership.** If the notes say "Sarah to follow up," the action item owner is Sarah.
