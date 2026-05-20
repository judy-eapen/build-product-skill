# Validate

Verify that what was built matches the PRD and designs. This is a quick smoke test after execution, before shipping.

## When to use

- After `/execute-plan` completes a phase or PR.
- Before demoing, shipping, or moving to the next phase.
- User will @mention the PRD and optionally the design catalog.

---

## Your process

### 1. Gather inputs

Ask:

- Which PRD? (e.g. `@~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`)
- Which Phase was just built?
- Is there a design catalog? (e.g. `@~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/design/[feature-name]-phase-1-designs.md`)
- Is the app running locally or deployed? (URL or localhost)

If the app is running, use the browser to visually verify. If not, review the code against the PRD.

### 2. Check acceptance criteria

For every user story in the target phase:

| User Story | Acceptance Criteria | Status | Notes |
|------------|-------------------|--------|-------|
| US-X.X | [AC from PRD] | Pass / Fail / Partial | [What's missing or wrong] |
| ... | ... | ... | ... |

Go through each AC literally. Don't interpret loosely. If the AC says "toast on failure," check that there's a toast on failure.

### 3. Check against designs (if available)

If a design catalog exists, compare each screen:

| Screen | Design Match | Notes |
|--------|-------------|-------|
| [Screen name] | Match / Partial / Mismatch | [What's different] |
| ... | ... | ... |

Focus on: layout, component usage, empty states, copy/messaging, mobile responsiveness.

### 4. Check non-functional requirements

From the PRD's NFR section, verify what's testable:

- [ ] Page loads within performance threshold
- [ ] Error states are handled (toast, inline, boundary)
- [ ] Loading states are present (skeleton, spinner)
- [ ] Mobile layout works at primary breakpoint
- [ ] Dark mode works (if specified)
- [ ] RLS / auth prevents unauthorized access (if applicable)

### 5. Produce the validation report

```markdown
# Validation Report: [App Name] — Phase [N]

**Validated:** [Date]
**PRD:** `[path]`
**Design Catalog:** `[path or N/A]`
**Overall:** Pass / Pass with notes / Fail

## Acceptance Criteria

| User Story | AC | Status | Notes |
|------------|-----|--------|-------|
| ... | ... | ... | ... |

## Design Comparison
[Table or "No design catalog provided"]

## Non-Functional Checks
- [x] / [ ] for each item

## Issues Found
1. [Issue]: [Description, severity, suggested fix]
2. ...

## Verdict
[Ready to ship / Fix N issues first / Needs significant rework]
```

### 6. Save

Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/validation/[feature-name]-phase-[N]-validation.md`.

---

## Important

- **Be literal.** Check exactly what the PRD says, not what you think it should say.
- **Don't scope-creep.** If something works but could be better, note it as "suggestion" not "fail."
- **Browser-verify when possible.** If the app is running, navigate it. Screenshots are evidence.
- **This is fast.** A validation should take minutes, not hours. Focus on the phase that was just built.
