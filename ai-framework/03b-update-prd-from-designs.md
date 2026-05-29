# Update PRD from Designs

After the design phase for a phase is **finalized** (design catalog complete, screens approved), update the PRD so it reflects what was actually designed. This keeps the PRD the single source of truth for execution and validation.

**When:** After `/design` for the phase is complete, before `/execute-plan`.

---

# What to Update

1. **Design catalog reference**
   - In the PRD section for that phase (e.g. Phase 1 Frontend Tasks or Phased Plan), add a line that points to the design catalog:
   - `**Design catalog:** ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-[N]-designs.md`

2. **Copy and flows that changed**
   - If any final copy, headings, or micro-flows differ from what the PRD originally said, update the PRD to match the built design (e.g. welcome message text, empty state copy, button labels).

3. **Acceptance criteria**
   - If design decisions added or changed behavior (e.g. "Consistency = votes / (habits × 7)", "Identity create is 3 steps"), add or adjust the relevant acceptance criteria so validation and execution use the same spec.

4. **Decision log**
   - Append an entry to the decision log sidecar `decisions/[feature]-decision-log.md` (NOT the PRD body — the PRD's § 10 is a pointer only) recording that the phase design was finalized and the design catalog is the reference:
   - e.g. `Phase [N] design finalized; design catalog [path]; PRD updated to match. [date]`

5. **Frontend task list (optional)**
   - If the design catalog introduced screens or flows not explicitly listed in the PRD frontend tasks, add a short bullet so the phase scope is clear for execution.

---

# What Not to Do

- Do not change scope (add new user stories or features) unless the user explicitly asks.
- Do not rewrite the whole PRD; only update the parts that need to align with the finalized designs.

---

# Output

- Updated PRD file (saved).
- Brief note: "PRD updated for Phase [N] from design catalog [path]. Ready for /execute-plan."
