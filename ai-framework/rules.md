# AI Product Framework Rules

These rules apply when doing product research, system design, or PRD creation in this repo.

## No Silent Assumptions
- Do not assume tech stack, auth, roles, data sources, or performance expectations.
- If the user does not know, propose 1–2 defaults, explain tradeoffs briefly, and wait for confirmation.

## Traceability
Every user-visible behavior must map to:
- a UI state change
- an API call (or local computation)
- a data read/write (if applicable)

## PRD Requirements
A PRD must include:
- Scope (in/out)
- Roles and permissions (and enforcement layer: FE/BE/both)
- Data model changes (fields, types, required/optional, validation, indexes)
- API contracts (request/response examples, errors, pagination, filtering, sorting)
- Non-functional requirements (performance, reliability, observability)
- Open questions
- Decision log — a one-line pointer only; the log itself is a sidecar at `decisions/[feature]-decision-log.md` (locked decisions with dates), never written inline in the PRD body. See `ai-framework/style-preferences.md` § Artifact Conventions.

## Phasing Rules
- Split work into independently shippable phases.
- Each phase must deliver user-visible value and be testable on its own.
- Each phase must include:
  - user stories
  - backend tickets
  - frontend tickets
  - acceptance criteria
  - definition of done
- Avoid forward dependencies. If unavoidable, explicitly label as dependency.

## Mongo/API Discipline (if applicable)
- For Mongo: specify indexes and backward compatibility implications.
- For search endpoints: specify filter semantics, pagination behavior, and default sort.
- Define error handling for upstream dependencies (timeouts, partial failures).

## Output Discipline
- Prefer structured output over prose.
- Use checklists and schemas where possible.
- If something is unknown, list it under Open Questions and ask the user.