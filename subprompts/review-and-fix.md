# Review and Fix PRD

Run both a product expert review and a CTO technical review on a PRD, then apply all findings directly to the PRD in one pass. This replaces the manual sequence of `/review-prd` + `/cto-review` + manually applying fixes.

## When to use

- After creating or updating a PRD, before design or execution.
- When you want a single command that reviews and fixes instead of three separate steps.

## Your process

### 1. Get the PRD

Ask the user to @mention the PRD file (e.g. `@~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`). Read it in full.

### 2. Run the product review

Follow the full process in `subprompts/review-prd.md`:
- Evaluate document quality (structure, problem/strategy, scope, user stories, clarity).
- Evaluate product quality (job-to-be-done, workflows, edge cases, onboarding, scope).
- Produce the structured review output.
- Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/product-review/[product-name]-prd-review.md`.

### 3. Run the CTO review

Follow the full process in `subprompts/cto-review.md`:
- Challenge the solution, consider alternatives.
- Evaluate architecture, scalability, security, phasing, risks.
- Produce the structured technical review output.
- Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/technical-review/[product-name]-cto-review.md`.

### 4. Collect all actionable findings

Read both review files. Extract every finding that requires a PRD change. Group them:

| Source | Finding | PRD section affected |
|---|---|---|
| Product review | [Finding] | [Section] |
| CTO review | [Finding] | [Section] |

### 5. Apply fixes to the PRD

Update the PRD file in place. For each finding:

- **User stories:** Add missing stories, add acceptance criteria, fix gaps.
- **Scope:** Add missing in-scope or out-of-scope items.
- **Data model:** Fix field types, add missing entities, add indexes.
- **API contracts:** Fix request/response schemas, add error cases.
- **Empty/error/loading states:** Add to user stories or AC if missing.
- **Security/permissions:** Add or fix role-based access, enforcement layer.
- **Non-functional requirements:** Add performance, observability, reliability items.
- **Decision log:** Add a row: "PRD reviewed (product + CTO) on [date]; findings applied."
- **Open questions:** Add any unresolved items from either review.

Do not add new features or expand scope beyond what the reviews recommend. Only fix what was identified.

### 6. Show the summary

After applying all fixes, output:

```
━━━ Review and Fix Complete ━━━

PRD: [path]
Product review: [path]
CTO review: [path]

Changes applied to PRD:
- [Change 1: e.g. "Added empty state AC to US-1.3"]
- [Change 2: e.g. "Fixed Session entity — added index on date field"]
- [Change 3: e.g. "Added security concern to Open Questions"]
- ...

Total: [N] changes applied.

PRD is updated and ready for /design or /execute-plan.
━━━
```

### 7. Save the PRD

Save the updated PRD file.

---

## Important

- **Run both reviews before applying fixes.** Do not apply product review fixes and then do the CTO review. Collect all findings first, then apply once.
- **Preserve the PRD structure.** Update sections in place; do not rewrite the whole document.
- **Be explicit about changes.** The summary must list every change so the user can verify.
- **Do not expand scope.** Only apply fixes that the reviews identified. If a review suggests a new feature, add it to Open Questions or Out of Scope unless the user confirms.
