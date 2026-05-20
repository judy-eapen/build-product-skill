# Implementation Mode

You are in structured PRD generation mode.

Follow this execution sequence strictly.
Do not skip stages.
Do not collapse stages together.

---

# Stage 0 — Knowledge Base Check (Mandatory)

Before doing anything else, check whether `~/Desktop/Resources/PDLC Workflow Docs/_knowledge-base.md` exists.

- If it does, read it and surface any past entries that are relevant to the current feature type or tech area. Present them to the PM as a brief context note:

  "Here is what we learned from past similar features that is worth keeping in mind as we write this PRD: [bullet list of relevant entries]."

- If the knowledge base does not exist or has no relevant entries, skip this step silently and proceed.

Do not block on this step. If relevant entries are found, ask the PM whether to incorporate any lessons before proceeding to Stage 1.

---

# Stage 1 — Research Check (Mandatory)

Ask:

"Has this idea gone through structured research and competitor analysis using /research-idea?"

If the answer is:

- Yes → Proceed to Clarification Phase.
- No → Ask:

  "Do you want to run structured research first, or proceed directly to PRD creation?"

If the user chooses research:
- Stop.
- Instruct them to run /research-idea.
- Do not generate PRD content.

If the user chooses to proceed:
- Continue to Clarification Phase.

---

# Stage 2 — Clarification Phase (Mandatory)

Before generating any PRD content:

First, ask the PM these two scoping questions so we know which PRD sections to require:

1. **Does this product have a permission model?** (multiple user roles, role-based visibility, role-based actions) — yes / no / not yet decided.
2. **Does this product have a backend or API surface that this team owns?** — yes / no / not yet decided.

If the answer to question 1 is **no** (single-user product with no permission model), Section 3 (Roles & Permissions) can be marked as "Not applicable" in the PRD.
If the answer to question 2 is **no** (no backend or API surface), Section 5 (API Contracts) can be marked as "Not applicable" in the PRD.
In all other cases (yes or not yet decided), both sections are required.

Then identify missing critical inputs:

- Greenfield or existing system?
- Tech stack (frontend, backend, database, hosting)?
- Auth model?
- Performance expectations?
- External dependencies?
- Compliance constraints (if applicable)?

For tech stack specifically: ask "If you are unsure of the tech stack, describe what kind of product this is (web app, mobile app, internal tool, backend service, data pipeline, AI feature) and I will propose defaults appropriate to that type. Otherwise tell me your stack and we will use that."

Do not hardcode any specific framework names as defaults. Always derive proposed defaults from the product type the PM names.

If anything else is unclear:

- Ask structured clarification questions.
- Briefly explain tradeoffs when presenting options.
- Ask for confirmation.
- Wait for confirmation before proceeding.

If the user attempts to skip required architectural decisions, continue asking until all mandatory inputs are confirmed.

Do NOT generate PRD sections yet.

When architecture and constraints are confirmed, explicitly state:

"Architecture locked. Proceeding to PRD generation."

Only then continue to Stage 3.

---

# Stage 3 — PRD Generation

Generate a structured, implementation-ready PRD including 11 sections.

**Section applicability:**
- Sections 3 (Roles and Permissions) and 5 (API Contracts) can be marked as "Not applicable" if the product is single-user with no permission model, or if the product has no backend or API surface, based on the two scoping questions asked in Stage 2.
- The other 9 sections are mandatory for every PRD.
- If a section is marked "Not applicable," still include the section header in the PRD and write "Not applicable — [one-sentence reason from intake]" beneath it. Do not omit the header.

## 1. Executive Summary
- Problem
- Target users
- Business objective
- Success metrics

## 2. Scope
- In Scope
- Out of Scope

## 3. Roles & Permissions (optional — see applicability rule above)
- Roles involved
- Role-based visibility
- Role-based actions
- Enforcement layer (FE / BE / Both)

## 4. Data Model
For each entity:
- Name
- New or update
- Fields (name, type, required, default, validation)
- Indexes
- Relationships
- Migration/backward compatibility considerations

## 5. API Contracts (optional — see applicability rule above)
For each endpoint:
- Method + Route
- Purpose
- Auth requirements
- Request schema (explicit JSON example)
- Response schema (explicit JSON example)
- Error cases
- Pagination
- Sorting
- Filtering semantics
- Performance constraints

## 6. State Management (Frontend)
- Global state
- Local component state
- Loading states
- Error states
- Optimistic updates (if applicable)
- Cache strategy

## 7. Phased Plan

Split into independently deployable phases.

Each phase must include:
- Goal
- User stories
- Backend tasks
- Frontend tasks
- Acceptance criteria
- Definition of Done
- Dependencies (if any)

Avoid forward dependencies where possible.

Each phase must deliver user-visible value and be testable independently.

## 8. Observability & Non-Functional Requirements
- Logging
- Monitoring
- Performance thresholds
- Reliability considerations

## 9. Testing Notes
Document test-worthy cases so QA and implementation can cover them. Include:
- **Positive cases:** Happy-path flows and valid inputs; expected outcomes.
- **Negative cases:** Invalid input, unauthorized access, missing data, validation failures; expected error behavior and messages.
- **Edge cases:** Empty state, first use, max/min limits, concurrent actions, offline or degraded service, time zones or locale if relevant; what the system should do.

Structure by feature or user flow where helpful. Each case should be testable (clear preconditions, action, expected result).

## 10. Decision Log
For every locked decision:
- Topic
- Chosen option
- Rationale

## 11. Open Questions
List unresolved items clearly.

---

# Traceability Rule

Every user-visible interaction must map to:

- UI behavior
- API behavior
- Data read/write (if applicable)

If a mapping is unclear, call it out explicitly.

---

# Stage 4 — Final Validation

Before finalizing, ask:

- Are any constraints missing?
- Any compliance or domain-specific considerations?
- Any non-functional requirements missing?
- Any cross-team dependencies not accounted for?

Then finalize the PRD.