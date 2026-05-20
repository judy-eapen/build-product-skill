# Generate Tests

Scaffold test cases from a user-stories breakdown (Gherkin AC) for the project's test framework. Use when:
- You have an approved user-stories breakdown and want executable test scaffolding.
- You want to ensure every Gherkin scenario has a corresponding test stub before implementation starts.
- You want to close the loop between documented AC and the test suite.

---

## Inputs

Collect these before generating:

**Required:**
1. **User stories breakdown** — path to `[feature-name]-user-stories.md`. Must contain Gherkin AC per story.
2. **Test framework** — which framework to generate for. Ask the PM if not known:
   - Frontend: Playwright, Cypress, Vitest, Jest, React Testing Library
   - Backend: Jest, Pytest, RSpec, Go test, JUnit
   - If "I don't know": ask for the tech stack and infer the standard framework for that stack.
3. **Test directory** — where tests should be written. Ask: "Where do your tests live? (e.g. `src/__tests__/`, `tests/`, `spec/`)"

**Optional:**
- **Codebase review** — if provided, extract existing test file patterns and naming conventions rather than guessing.

---

## Execution

### Step 1 — Discover test conventions

If a codebase review is provided, read it for:
- Existing test file naming patterns (e.g., `*.test.ts`, `*.spec.ts`, `_test.go`)
- Import patterns (e.g., `describe`/`it`, `test`, `def test_`, `func Test`)
- Assertion libraries in use (e.g., `expect`, `assert`, `should`)
- Any custom test helpers or fixtures referenced

If no codebase review is provided, use the standard conventions for the named framework.

### Step 2 — Map stories to test files

For each story in the breakdown:
- Determine the appropriate test file name from the story title and type (FE → component/page test; BE → service/API test).
- Group related stories under the same test file if they test the same component or endpoint.
- Output a file map:

```
US-1.1 (FE) → src/__tests__/SavedSearches.test.tsx
US-1.2 (BE) → tests/api/savedSearches.test.ts
US-2.1 (FE) → src/__tests__/FilterPanel.test.tsx
...
```

Present the file map to the PM and ask for confirmation before generating files. Allow renaming.

### Step 3 — Generate test scaffolding

For each test file, generate scaffold code:

**Rules:**
- One `describe` block per story (or one `test class` / `test module` per story, depending on framework).
- One test stub per Gherkin scenario. The stub name is the scenario label verbatim (or closest equivalent).
- Each stub body contains: a comment with the full Gherkin (`Given / When / Then`), then `// TODO: implement` (or language equivalent).
- Do NOT implement the test logic — only scaffold. Implementations come from the engineer.
- Do NOT import components or modules that do not yet exist. Use placeholder import comments: `// TODO: import [ComponentName] from '[path]'`
- Include any `beforeEach` / `afterEach` stubs that the Gherkin scenarios imply (e.g., "Given I am logged in" → `beforeEach` stub for auth setup).

**Example output for Playwright (TypeScript):**

```typescript
// US-1.1 — View saved searches
// Generated from user-stories breakdown — do not delete scenario labels

import { test, expect } from '@playwright/test';
// TODO: import page objects or helpers as needed

test.describe('View saved searches', () => {

  test.beforeEach(async ({ page }) => {
    // TODO: Given I am a logged-in user
    // Set up auth state here
  });

  test('User sees list of saved searches on dashboard', async ({ page }) => {
    // Given I am on the dashboard
    // When I navigate to the Saved Searches section
    // Then I see a list of my saved searches
    // And each search shows name, date saved, and result count
    // TODO: implement
  });

  test('Empty state shown when no searches saved', async ({ page }) => {
    // Given I have no saved searches
    // When I navigate to the Saved Searches section
    // Then I see the empty state message "No saved searches yet"
    // And I see a CTA to create my first search
    // TODO: implement
  });

});
```

### Step 4 — Write files

Write each generated test file to the path confirmed in Step 2.

Print a confirmation line per file:
```
✓ Generated: [file path] — [N] test stubs ([story ID])
```

### Step 5 — Summary

After all files are written, output:

```
━━━ Test Scaffolding Complete ━━━

Framework: [name]
Stories covered: [N]
Test files generated: [N]
Total stubs: [N]

Files:
- [path] — [N] stubs
- [path] — [N] stubs
...

Next steps:
- Review the file map — rename files if needed.
- Implement each TODO stub during the execution phase.
- Run the test suite to confirm all stubs compile/parse (they should fail, not error).
━━━
```

---

## When to run

- **In the Work pipeline:** Run after Gate 3 (user stories breakdown approved) and before handing off to the engineering team. The scaffolding ships alongside the Jira tickets as a starting point for the assigned engineer.
- **In the Personal pipeline:** Run after the user stories breakdown is finalized (between Gate 1 and execution). Engineers implement against pre-existing stubs.
- **Standalone:** Run `/generate-tests` any time with a breakdown file and framework name.

---

## Important

- Never implement test assertions — only scaffold. Implemented tests are the engineer's responsibility.
- Never delete existing test files. If a file already exists at the target path, append new `describe`/`test` blocks below the existing content and add a comment: `// Added by generate-tests on [date] — review for overlap with existing tests`.
- Never modify non-test files.
