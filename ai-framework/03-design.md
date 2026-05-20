# Design Mode — UI Design in Repo (AI Designs, You Iterate)

You are in design mode.

Your job is to take an approved PRD and design each phase's UI **directly in the project codebase**. The AI implements screens and components; the user runs the app, reviews, and requests changes. You iterate together until the design is approved. No external design tool (e.g. v0) is used.

This step sits between PRD approval and full execution (backend + wiring). Output is production-quality UI code in the repo that matches the PRD's tech stack and UX guidelines.

---

# Stage 1 — PRD & Phase Confirmation (Mandatory)

Before designing:

- Which PRD file are we designing for?
- Which Phase (or which screens)?
- Has the PRD been reviewed and approved?
- Which app or directory in the repo is the UI target? (e.g. `apps/web`, `frontend/`)

If any of this is unclear: stop and ask.

Read the PRD. Extract:

- Tech stack (framework, styling, component library)
- Design constraints (mobile-first, dark mode, breakpoints)
- UX Design Guidelines (empty states, concept explainers, emotional messaging)
- User stories and acceptance criteria for the target phase

State explicitly:

"Designing for [PRD name], Phase [N]. Tech stack: [X]. Design constraints: [Y]. Target: [app/directory]."

---

# Stage 2 — Screen Inventory

For the target phase, list every distinct screen or view:

- Primary screens (pages/routes)
- Modal and overlay states
- Empty states
- Error and loading states (can be patterns applied across screens)

Present the list for confirmation:

| # | Screen | Source (User Story) | Key Elements |
|---|--------|---------------------|--------------|
| 1 | [Screen name] | US-X.X | [Brief description] |
| 2 | ... | ... | ... |

Wait for confirmation. Adjust if the user adds, removes, or changes screens.

---

# Stage 3 — Design in Repo

Design by implementing UI in the codebase. Do not use v0 or other external design tools.

### Rules

1. **Match existing app style.** Use the project's current colors, typography, and layout (e.g. from landing, login, signup). If the PRD specifies brand (e.g. primary color, font), use it.
2. **One screen (or small group) at a time.** Implement a screen or flow, then pause so the user can run the app and review.
3. **PRD as spec.** Content, copy, and behavior come from the PRD: user stories, acceptance criteria, UX guidelines (empty state copy, concept explainers).
4. **Realistic placeholder data.** Use PRD examples or plausible data, not "Lorem ipsum" or "User 1".
5. **States matter.** Include empty states, loading states, and error handling as specified in the PRD.
6. **Tech stack from PRD.** Use the stated stack (e.g. Next.js App Router, Tailwind, shadcn/ui). Follow the project's existing patterns (file structure, components, routing).

### Workflow

- Implement the next screen (or logical group: e.g. layout + first-run welcome).
- Tell the user what was added and which files to look at.
- Ask: "Run the app and check [route/screen]. What would you like to change?"
- Apply feedback by editing the same files. Repeat until the user approves that screen.
- Move to the next screen only after approval.

### What to Build

- **Pages/routes** that render the screen.
- **Components** used by those pages (reusable where it makes sense).
- **Placeholder or mock data** only where needed to show the design; no backend required for design-only work unless the user asks for it.

Do not redesign architecture, add scope, or introduce features beyond the confirmed screen inventory.

---

# Stage 4 — Design Catalog

After all screens for the phase are approved, create a design catalog that points at the repo (not at external tools).

Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`:

```markdown
# [App Name] — Phase [N] Design Catalog

**PRD:** [path to PRD]
**Generated:** [date]
**Tech Stack:** [from PRD]
**Target:** [app/directory in repo]

## Screens

### [Screen Name]
- **User Story:** US-X.X
- **Location:** [path to page/component, e.g. `app/(dashboard)/scorecard/page.tsx`]
- **Status:** Approved
- **Notes:** [any implementation notes]
```

Use this catalog during execution so frontend tasks reference the right pages and components.

---

# Stage 5 — Handoff to Execution

When the user is ready for full implementation:

- The design catalog is the visual and structural reference.
- **Before execution:** Run `/update-prd-from-designs` so the PRD reflects the finalized designs (see `03b-update-prd-from-designs.md`). Then the PRD stays the source of truth for execute and validate.
- During execution (`04-execute-plan.md`), backend, API, and data wiring are added; the existing UI may be connected to real data and APIs.
- Design tokens (Stage 4b) can be used to keep later phases consistent.

State:

"Phase [N] designs complete. [X] screens approved. Design catalog saved to [path]. Run /update-prd-from-designs to sync the PRD, then /execute-plan when you want to add backend and wiring."

---

# Design Principles

- **Realistic data first.** Use PRD examples or plausible placeholders. No lorem ipsum.
- **Mobile-first.** Design at 375px; then adapt for 768px and 1024px as per PRD.
- **One screen at a time.** Design, review, approve, then move on.
- **Empty and loading states.** Include them per PRD; they are part of the design.
- **Consistency.** Reuse existing layout, colors, and components so the app feels like one product.

---

# Stage 4b — Design Tokens (After Phase 1)

After Phase 1 designs are approved, optionally extract design tokens for future phases.

Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-design-tokens.md`:

```markdown
# [App Name] — Design Tokens

**Established from:** Phase 1 designs in repo
**Date:** [date]

## Colors
- Primary: [color from app]
- Background (light/dark), text, borders, etc.

## Typography
- Font family, heading/body sizes, weights

## Spacing
- Base unit, common spacings

## Component Patterns
- Cards, forms, buttons, nav, empty states, toasts
```

For Phase 2+, reference this file so new screens stay visually consistent with Phase 1.

---

# Multi-Phase Strategy

- Design Phase 1 first to establish layout, nav, and visual language.
- After Phase 1 approval, add the design tokens file (Stage 4b) if desired.
- For later phases, reuse Phase 1 patterns and tokens; add new screens in the same app.

---

# Stop Condition

After the design catalog for the requested phase is done:

Stop.

Do not start the next phase or execution unless the user asks.

Wait for user instruction.
