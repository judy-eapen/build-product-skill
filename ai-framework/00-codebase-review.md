# Codebase Review

Runs after research and before the PRD is written. Grounds the PRD in the codebase from the first line.

Read `ai-framework/rules.md` and `ai-framework/error-handling.md` before executing.

---

## Step 1 — Input Collection

Ask the PM to do one of four things:

1. **Point to folders on disk** (preferred). Paste the absolute path(s) to the frontend codebase and/or backend codebase — for example:
   - Frontend: `/Users/[name]/Desktop/my-app-frontend/`
   - Backend: `/Users/[name]/Desktop/my-app-backend/`
   - Monorepo (both in one folder): `/Users/[name]/Desktop/my-app/`

   The skill will read directly from those paths. The PM typically gets these by downloading the repo as a zip from GitHub and extracting to Desktop, or pointing at an existing local clone.

2. **Paste code inline** — for specific files or snippets when folder access isn't practical.

3. **Describe the architecture in plain English** — when code isn't accessible at all (frameworks, services, data stores, key modules, integration points).

4. **Confirm greenfield** — no existing codebase.

Do not proceed to Step 2 until one of these four responses is received. If the response is ambiguous, ask again. Do not infer.

### If folder paths were provided

Before moving to Step 2, do the following with Bash and Read tools:

1. **Confirm each path exists**: `ls -la [path]`. If a path is missing, ask the PM to correct it.
2. **Get the directory shape**: list top-level files and folders. Use `tree -L 2 [path]` or `find [path] -maxdepth 2 -type d` if `tree` isn't available.
3. **Identify the stack from manifest files**: read whichever of these exists — `package.json`, `requirements.txt`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `Gemfile`, `composer.json`, `pom.xml`, `build.gradle`, `mix.exs`, `pubspec.yaml`. The manifest tells you the language, framework, and dependencies in seconds.
4. **Read the README** if one exists at the root.
5. **Sample 5–10 representative files** that match the feature being planned. Look for: main entry points, core domain modules, database schemas or models, API route definitions, key shared components. Read enough to understand patterns — don't try to read everything.
6. **Note the size**: report the total file count and rough breakdown (e.g., "FE folder: 412 files, ~80 components; BE folder: 78 files, ~20 endpoints"). This helps the PM understand what was and wasn't reviewed.
7. **Explicitly list what was NOT reviewed**: large directories you skipped (e.g., `node_modules/`, `dist/`, `migrations/`, test fixtures). Surface as "unreviewed areas" in the output.

If both a frontend folder and a backend folder are provided, evaluate them separately in the output (one set of findings for FE, one set for BE), then a brief cross-cutting note about how they interact (shared types, API contracts, etc.).

---

## Step 2 — Codebase Analysis

### Path A — Code or architecture was provided (including folder paths)

Produce a structured output with exactly four sections:

**1. What already exists that can be reused**
- For each component, module, or pattern: name + one line on how it applies to this feature.

**2. What needs to be built new**
- For each piece: name + brief reason why it cannot be reused.

**3. Patterns and conventions to follow**
- Naming, structure, and architecture patterns already in use that new work should match.

**4. Technical risks and assumptions**
- For each risk: description + impact rating (HIGH / MEDIUM / LOW) + mitigation suggestion.

### Path B — No codebase provided (greenfield)

Produce a single section only:

**Risk Flags — Greenfield Assumptions**

List the top 3 to 5 assumptions the PRD will be making about the technical environment. Flag each one explicitly as an assumption that needs confirmation before build begins.

Example format:
- ASSUMPTION: [statement]. Needs confirmation before PRD lock.

---

## Step 3 — Output

Confirm the file path to the PM before writing:

```
Writing: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/codebase-review/[feature-name]-codebase-review.md
```

Then write the structured findings to that path.

If the parent folders do not exist, create them automatically.

---

## Step 4 — Handoff Note

Produce a one-paragraph summary of the most important codebase finding. Keep it tight (3 to 5 sentences). This paragraph is passed as context to the PRD step so the PRD is grounded in the codebase from the first line.

The full codebase review output file is also passed to the Technical Review agent later.

State explicitly:

"Codebase review complete. Handoff summary: [paragraph]. Proceeding to PRD."

---

## Rules

- Never assume anything about the tech stack that is not visible in the provided code or stated by the PM.
- Never suggest architectural changes to existing code. Only identify what is relevant to the new feature.
- **Folder paths:** never modify files in the codebase. The folders are read-only inputs to the review. The skill should never `Edit`, `Write`, or otherwise touch files under the PM's source folders.
- **Folder paths:** if either path is invalid, ask the PM to correct it before proceeding. Do not silently work with one folder and skip the other.
- **Folder paths:** be honest about what was and wasn't reviewed. If a folder has 1,000 files and you sampled 10, say so explicitly. Sampled findings are best-effort, not exhaustive.
- If the PM pasted partial files, explicitly flag which areas were not reviewed and label them as unreviewed assumptions.
- If the PM described architecture in plain English instead of pasting code, every finding must be conditional on that description being accurate. Note this in the output.
- Do not expand scope. The output is a reference document, not a redesign.
