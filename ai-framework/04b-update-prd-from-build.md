# Update PRD from Build

After a phase is **built and validated** (execute-plan done, /validate run), update the PRD so it reflects what was actually implemented. This keeps the PRD the single source of truth for the next phase and for future validation.

**When:** After `/validate` for the phase (workflow step 8b).

---

# What to Update

1. **Data model**
   - Add or change entities, fields, indexes, and relationships to match the migrations and code (e.g. `scorecard_entries.identity_id`, new `habits` table).

2. **Implementation notes**
   - If something was built differently from the original spec (without changing scope), document it in the PRD (e.g. "Stack is shown per-habit as 'After [X]'; no separate chain diagram in Phase 2").

3. **Scope or AC that changed**
   - If a feature was removed, deferred, or simplified during execution, update the phase's Frontend/Backend tasks and Acceptance Criteria so they match what exists (e.g. "Design-review 'View empty state' toggle removed; empty state still shown when list is empty").

4. **Copy or flows**
   - If final UI copy or flows differ from the PRD (e.g. identity creation now includes linking habits), update the relevant bullets or AC.

5. **Decision log (optional)**
   - Add a row if the build introduced an important decision (e.g. "Phase 2 stack: per-habit label only; full chain diagram deferred.").

---

# What Not to Do

- Do not add new user stories or features; only sync the PRD to what was built.
- Do not rewrite the whole PRD; only update the parts that need to align with the built product.

---

# Output

- Updated PRD file (saved).
- Brief note: "PRD updated for Phase [N] from build. Ready for next phase."
