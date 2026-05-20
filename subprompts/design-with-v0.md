# Design with v0

Generate UI designs using v0 (vercel.com/v0). You produce detailed v0 prompts from the PRD; the user generates designs in v0 and brings them back for review.

For in-repo design (AI builds screens directly in code), use `/design` instead.

---

## Stage 1 -- PRD & Phase Confirmation (Mandatory)

Before designing:

- Which PRD file are we designing for?
- Which Phase (or which screens)?
- Has the PRD been reviewed and approved?
- Which app or directory in the repo is the UI target?

If any of this is unclear: stop and ask.

Read the PRD. Extract:

- Tech stack (framework, styling, component library)
- Design constraints (mobile-first, dark mode, breakpoints)
- UX design guidelines (empty states, concept explainers, emotional messaging)
- User stories and acceptance criteria for the target phase

State explicitly:

"Designing for [PRD name], Phase [N]. Tech stack: [X]. Design constraints: [Y]. Target: [app/directory]."

---

## Stage 2 -- Screen Inventory

For the target phase, list every distinct screen or view:

- Primary screens (pages/routes)
- Modal and overlay states
- Empty states
- Error and loading states

Present the list for confirmation:

| # | Screen | Source (User Story) | Key Elements |
|---|--------|---------------------|--------------|
| 1 | [Screen name] | US-X.X | [Brief description] |
| 2 | ... | ... | ... |

Wait for confirmation. Adjust if the user adds, removes, or changes screens.

---

## Stage 3 -- Generate v0 Prompts

For each confirmed screen, generate a detailed v0 prompt. Each prompt must include:

### Content from the PRD
- Page title and navigation context (where does this screen sit in the app?)
- Content and copy from user stories and acceptance criteria
- Data to display (use realistic placeholder data from the PRD, not "Lorem ipsum")
- Actions the user can take (buttons, forms, interactions)

### Layout and structure
- Component hierarchy (page > sections > cards/lists > items)
- Key UI patterns (table, card grid, form, dashboard, empty state)
- Responsive behavior (mobile-first breakpoint, tablet, desktop)

### States
- Default state (with data)
- Empty state (zero items, first-time user)
- Loading state
- Error state (if specified in PRD)

### Styling constraints
- Framework: [from PRD, e.g. "Next.js App Router with Tailwind CSS and shadcn/ui"]
- Color scheme, fonts, or brand guidelines (if specified in PRD)
- Dark mode (if specified)
- Match existing app style (if this is a later phase, reference Phase 1 design tokens or existing screens)

### v0 prompt format

Output each prompt in a code block the user can copy-paste into v0:

```
Create a [page type] for [app name].

[Description of what the page shows and what the user does here.]

Tech stack: [framework, styling, component library].

Layout:
- [Component 1]: [what it contains]
- [Component 2]: [what it contains]
- ...

Data to display:
- [Field 1]: [example value]
- [Field 2]: [example value]
- ...

Actions:
- [Button/interaction 1]: [what it does]
- ...

States:
- Empty state: [what to show when there is no data]
- Loading: [skeleton/spinner]

Styling:
- [Color, font, dark mode, responsive constraints]
```

Generate **one prompt per screen** (or per logical group if screens are closely related, e.g. a list page + its detail modal).

After presenting a prompt, ask: "Copy this into v0 and share the result. I'll review it against the PRD."

---

## Stage 4 -- Review v0 Output

When the user shares v0 output (screenshot, URL, or code):

1. Compare against the PRD acceptance criteria for that screen.
2. Check: correct copy/content, correct actions, correct empty/error states, responsive layout, styling consistency.
3. If something is missing or wrong, state what and suggest a follow-up v0 prompt to fix it.
4. If the screen matches the PRD, approve: "This screen matches the PRD. Ready to move on."

Repeat for each screen until all are approved.

---

## Stage 5 -- Design Catalog

After all screens for the phase are approved, create a design catalog.

Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`:

```markdown
# [App Name] -- Phase [N] Design Catalog

**PRD:** [path to PRD]
**Generated:** [date]
**Tech Stack:** [from PRD]
**Design Tool:** v0
**Target:** [app/directory in repo]

## Screens

### [Screen Name]
- **User Story:** US-X.X
- **v0 Source:** [v0 URL or "user-provided screenshot"]
- **Status:** Approved
- **Notes:** [any implementation notes or deviations from v0 output]
```

---

## Stage 6 -- Handoff to Execution

When the user is ready for implementation:

- The design catalog is the visual and structural reference.
- Run `/update-prd-from-designs` to sync the PRD with the finalized designs before execution.
- During `/execute-plan`, v0-generated code can be used as a starting point; the AI adapts it to the project's file structure and patterns.

State:

"Phase [N] designs complete. [X] screens approved via v0. Design catalog saved to [path]. Run `/update-prd-from-designs` to sync the PRD, then `/execute-plan` when ready."

---

## Stop Condition

After the design catalog for the requested phase is done:

Stop.

Do not start the next phase or execution unless the user asks.

Wait for user instruction.
