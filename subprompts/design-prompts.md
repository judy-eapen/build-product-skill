# Screen Design Prompts — v0 / Figma Make

Generates design prompts for every screen in the current phase. The same prompts work in both v0 and Figma Make — the PM chooses the tool. Figma Make has the additional advantage of being able to reference your team's existing Figma design system components directly.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 0 — Input Check

Before doing anything else, confirm you have:
1. The approved PRD (post-Gate 1) at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`
2. The system design doc (if produced) at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[feature-name]-system-design.md`
3. The visual diagram (if produced) at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/diagrams/[feature-name]-feature-diagram.md`

If running standalone: ask the PM for the PRD file path or phase scope.

Ask the PM:
1. **Which phase are we designing?** (e.g. Phase 1, Phase 2)
2. **Which tool will you use — v0 or Figma Make?**
   - Both tools take the same prompt format.
   - Figma Make: can reference your team's Figma component library by name (e.g. "use the Button/Primary component from our design system").
   - v0: generates components from scratch.
3. **Figma Make only:** Do you have a Figma design system or component library to reference? If yes, what are the key component names or the library file URL?

---

## Step 1 — PRD Confirmation

Read the PRD. Confirm back to the PM:
1. The phase scope: which user stories are in scope for this phase.
2. The screens needed: a numbered list of every distinct screen the phase requires.
3. Any design constraints mentioned in the PRD (branding, accessibility requirements, tech stack notes).

Ask: "Does this match your expectations? Any screens to add or remove before I generate prompts?"

Wait for confirmation. Adjust if needed.

---

## Step 2 — Screen Inventory

Build a complete screen inventory for the phase. For each screen:

| # | Screen name | User story(s) | States needed | Notes |
|---|-------------|---------------|---------------|-------|
| 1 | [name] | [US-ID(s)] | Empty / Loading / Error / Populated | [any constraints] |

Every screen must trace to at least one user story from the PRD. Do not invent screens that are not in scope.

If the visual diagram was produced, cross-check: every flow node in the diagram should appear as a screen here.

Present the inventory to the PM. Ask: "Any screens missing or out of scope?"

---

## Step 3 — Generate Prompts

Generate one design prompt per screen. Each prompt must be self-contained — the PM pastes it directly into v0 or Figma Make without needing to add context.

### Prompt structure (repeat for each screen)

```
---
Screen: [Screen Name]
User story: [US-ID — one-line summary]

Design a [screen type: page / modal / drawer / component] for [product name].

Context:
[2-3 sentences describing what this screen does, who uses it, and when they see it.
Use the PRD's problem statement and user personas.]

Layout:
[Describe the overall layout — e.g. "Full-page with a sidebar navigation, main content
area, and a top header."]

Content:
[List every element that must appear on this screen, in order from top to bottom:
- [Element 1]: [description]
- [Element 2]: [description]
... include all buttons, labels, inputs, lists, empty states, error messages]

States:
- Default / Populated: [what the screen looks like with real data]
- Empty state: [what the screen looks like with no data — exact copy/message if specified in PRD]
- Loading state: [skeleton structure or spinner behavior]
- Error state: [what appears if the action fails — exact error copy if specified in PRD]

[Figma Make only — omit if using v0:]
Component references:
- Use [Component/Name] for the primary action button.
- Use [Component/Name] for the navigation sidebar.
- Use [Component/Name] for form inputs.
[List any other design system components to use]

Interactions:
[Describe key interactions — e.g. "Clicking 'Save' shows a success toast and returns
to the list view." Only include interactions scoped in this phase.]

Constraints:
[Any specific requirements: responsive layout, accessibility notes, color/brand notes]
---
```

Generate all prompts before presenting. Do not pause between screens.

---

## Step 4 — PM Review

Present all generated prompts to the PM. Ask:

"Review the prompts above. For each screen:
- Does the content list cover everything from the PRD?
- Are the states correct?
- Are the component references accurate (Figma Make only)?

Reply with any corrections screen by screen, or say 'approved' to proceed to the design catalog."

Apply any corrections the PM gives. Do not re-generate all prompts — update only the screens flagged.

---

## Step 5 — Design Catalog

After the PM approves the prompts (or returns from v0 / Figma Make with outputs):

Ask the PM:
1. **v0:** "Paste the v0 preview URL(s) for each screen, or describe what was generated so I can document it."
2. **Figma Make:** "Share the Figma file URL (or frame link for each screen) so I can log it in the design catalog."

Write the design catalog to:

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md
```

Confirm the file path before writing: `Writing: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`

Catalog structure:

```markdown
# Design Catalog — [Feature Name] Phase [N]

**Tool used:** [v0 / Figma Make]
**Phase:** [N]
**Date:** [YYYY-MM-DD]
**PRD:** [path]

## Screens

### [Screen Name]
- User story: [US-ID]
- Prompt: [paste the prompt used]
- Output URL: [v0 preview URL or Figma frame URL]
- States covered: [list]
- Notes: [any deviations from the PRD, decisions made, copy changes]

[Repeat for each screen]

## Design tokens (Phase 1 only)
[Color palette, typography, spacing scale — derived from the generated designs or the
team's Figma design system]

## Open design questions
[Any screens where the prompt didn't fully resolve the design — surface these as open
questions for the PM]
```

---

## Rules

- **Same prompt, two tools.** The prompt structure above works identically in v0 and Figma Make. The only difference is the "Component references" block, which is Figma Make-specific.
- **No scope creep.** Every screen must trace to the PRD. If the PM asks for a screen that is not in the PRD, say: "That screen isn't in the current PRD scope. Should we add it? If yes, run `/change-mode` to add it properly before designing it."
- **State coverage is mandatory.** Every screen needs at minimum a default/populated state and an empty state. If the PRD doesn't specify the empty state copy, use sensible defaults and note them in the catalog.
- **Figma Make advantage.** If the team has a Figma design system, Figma Make will produce more consistent outputs than v0 because it can reference existing components. Mention this when the PM chooses the tool.
- **Do not implement.** This skill generates prompts and catalogs designs. It does not write code. Implementation is handled by the engineering team from the Jira tickets.
