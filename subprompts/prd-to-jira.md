# PRD to Jira

## Primary input: the User Stories Breakdown

When this prompt runs as Step 11 of the Work pipeline, the **primary input is the user-stories breakdown** at:

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/user-stories/[feature-name]-user-stories.md
```

This file has already been approved at Gate 3. It contains every story with FE/BE labels, exhaustive Gherkin acceptance criteria, sequencing, sizing, testing notes, and linked-pair relationships. Read this file first.

**Fallback:** if the breakdown file does not exist (e.g., running this prompt standalone outside the pipeline), read the PRD directly at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md` and infer FE/BE labels from the Phased Plan.

When the breakdown exists, do not re-infer FE/BE, do not re-write Gherkin, do not re-derive sequencing. The breakdown is the source of truth.

---

## Read intake conventions before composing any ticket

Read `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/_pipeline-state.json` and pull `intake.jira_ticket_conventions`. This is the free-text the PM gave at intake describing every per-ticket convention their team applies.

Interpret it carefully. Look for:

- **Labels** to apply to every ticket (e.g., pod tag, area tag). Add them to the `labels` array on every Story and the Epic.
- **Title format** rules. Apply them when composing the `summary` field. For example, "verb-first, `[BE]`/`[FE]` prefix" means a BE story becomes `[BE] Create notes endpoint` and an FE story becomes `[FE] Add note from listing card`.
- **BE/FE split rule.** If the convention says BE and FE are always separate tickets, never combine them — even if the breakdown shows a single story that covers both, split it. If the convention says they're combined, do not split linked pairs.
- **Custom field defaults.** If a field like "Testable" has a stated default (e.g., "always Yes"), set it on every ticket. Look up the custom field ID via `getJiraIssueTypeMetaWithFields` if you don't have it cached.
- **Fields to leave blank.** If the convention says (e.g.) "don't fill in Story Points," omit that field from the payload entirely. Do not guess.
- **Link conventions.** If the convention says (e.g.) "link BE↔FE pairs with 'Relates to'," do that explicitly after creating each pair. If it says "use 'Blocked by' with a note," apply that during sequencing.
- **Anything else.** Conventions you don't recognize: ask the PM once before proceeding rather than guessing.

If `intake.jira_ticket_conventions` is empty, missing, or says "we don't have specific conventions yet," fall back to sensible defaults (no special labels beyond what's on the breakdown stories, plain summary text, do not pre-fill custom fields).

If the conventions are ambiguous or conflict with the breakdown (e.g., conventions say "combined BE+FE" but the breakdown has them split): surface the conflict to the PM and ask which to follow. Do not silently overrule the breakdown.

---

## Knowledge Base Sizing Check

Before creating tickets, check whether `~/Desktop/Resources/PDLC Workflow Docs/_knowledge-base.md` exists.

- If it does, read the sizing accuracy entries from past features. Look for a consistent pattern of undersizing or oversizing a particular type of work.
- If a pattern exists, surface it to the PM as a warning. Ask whether to adjust the size labels from the breakdown up or down based on the pattern. (The breakdown's sizes are starting points; the knowledge base may suggest adjustments.)
- If no pattern exists or the knowledge base does not exist, skip silently.

---

## Pre-flight: deduplication check

Before creating any tickets, check whether tickets for this feature already exist in Jira. Running the pipeline twice without this check creates duplicate tickets.

**Step 1 — Check the manifest file.**

If `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-manifest.md` exists and contains issue keys, a prior run has already created tickets.

Read the manifest. For each US-ID with a Jira issue key, call `getJiraIssue` to confirm the ticket still exists.

- If tickets exist and look correct: present a diff to the PM:
  ```
  ⚠ Existing Jira tickets found for this feature:
  - [N] tickets already created (listed in manifest)
  - [N] new stories in current breakdown not yet in Jira

  Options:
  1. Skip existing, create only new ones (safe default)
  2. Update existing tickets with current breakdown content, create new ones
  3. Create all tickets again (will create duplicates — use only if you deleted the old ones)

  Which option? (1 / 2 / 3)
  ```
  Wait for PM decision before proceeding.

- If manifest exists but all tickets return 404 (deleted): proceed with full creation.
- If manifest does not exist: proceed with full creation.

**Step 2 — Check Jira directly (if no manifest).**

If no manifest exists, query Jira for any existing Epic whose summary matches the PRD title:
```
searchJiraIssuesUsingJql: project = [PROJECT] AND issuetype = Epic AND summary ~ "[feature name]"
```
If a matching Epic is found, surface it to the PM:
```
⚠ An Epic matching "[feature name]" already exists in Jira: [EPIC-KEY] — [summary]
  Created: [date]

  Do you want to add stories to this existing Epic, or create a new one?
  (existing / new)
```
Wait for PM decision before proceeding.

---

## Pre-flight: write creation manifest before bulk creation

Before creating any tickets, write a pre-creation manifest at:
```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-manifest.md
```

Use this format for each story that will be created:

```markdown
| US-ID | Title | Type | Jira issue key | Jira URL | Status |
|---|---|---|---|---|---|
| US-1.1 | [title] | FE | — | — | Pending |
| US-1.2 | [title] | BE | — | — | Pending |
```

**Update the manifest row-by-row as each ticket is created.** Do not wait until all tickets are done. This makes the creation transactional: if the process is interrupted, re-running detects which tickets were created (via the deduplication check above) and resumes from the first `Pending` row.

Replace `—` with the actual issue key and URL as each ticket succeeds. Change `Status` to `Created` or `Failed` after each attempt.

---

## Pre-flight: validate custom field IDs

Custom field IDs are Jira-instance-specific. Before bulk creation, validate them. Fail fast if wrong.

1. Use `getJiraIssueTypeMetaWithFields` (or equivalent) to list the available custom fields for the intake-supplied Jira project + Story issue type.
2. Identify the field IDs for: User Story narrative, Acceptance Criteria, Testable (yes/no), and any team / role fields the PM specified at intake.
3. If any required custom field cannot be located, ask the PM to provide the field ID before proceeding. Do not guess.
4. Test the IDs against ONE field on ONE issue (e.g., create a dry-run draft, then delete) before bulk creation if the MCP allows.

If validation fails, write the entire intended export to the local fallback file (Error Type 4) and notify the PM. Do not create partial tickets.

---

Create Jira Stories and one or more Epics from the approved user-stories breakdown. Create tickets in the **Jira project provided at intake** (see Intake Parameters in `CLAUDE.md`). Fill the fields the PM uses when creating tickets manually.

## When to use

- Step 11 of the Work pipeline: input is the approved breakdown (Gate 3 already passed).
- Standalone: a PM has a PRD and wants Jira tickets directly. In this case, infer from the PRD's Phased Plan.
- Default: create tickets in the **Jira project name** the PM provided at intake, issue type **Story**. If the PM did not provide a project name, ask before creating any tickets.

---

## Multi-epic mode — reading the epic grouping from state

As of v2.4.0, the User Stories Breakdown step (`ai-framework/06-user-stories.md`) produces a multi-epic grouping. Read it from `_pipeline-state.json` → `user_stories.epics[]`:

```json
"epics": [
  { "epic_id": "E1", "title": "Saved Searches - Listing",   "phase": 1, "theme": "...", "story_ids": ["US-1.1", "US-1.2"] },
  { "epic_id": "E2", "title": "Saved Searches - Management", "phase": 1, "theme": "...", "story_ids": ["US-1.3", "US-1.4"] }
]
```

**Create one Jira Epic per entry**, then create the listed stories under that Epic via `parent` = the new Epic's key.

If `user_stories.epics` is missing or empty (pre-v2.4.0 breakdown), fall back to single-Epic mode: create one Epic from the PRD title and put every story under it.

### Existing-Epic detection per group

Before creating each Epic, query Jira for any existing Epic in the project whose summary matches the proposed title:

```
searchJiraIssuesUsingJql: project = [PROJECT] AND issuetype = Epic AND summary ~ "[epic title]"
```

If a matching Epic exists, surface it to the PM:

```
⚠ An Epic matching "[epic title]" already exists in Jira: [EPIC-KEY] — [summary]
  Reuse this Epic for the [N] stories under it, or create a new one with a suffix?
```

Default to reuse. The PM can override per-Epic.

### DRAFT story labeling

Read `_pipeline-state.json` → `user_stories.draft_stories[]`. For every story listed there, add a `draft` label (in addition to the usual frontend/backend labels) when creating the Jira ticket. This makes the DRAFT subset queryable in Jira via `labels = "draft"`. When `/change-mode` → "Designs arrived" refreshes a story, it removes this label.

---

## Epic description — what to write (and what NOT to write)

Every Epic created from the multi-epic grouping needs its own description. The Epic description is what every stakeholder sees first when they open the Epic in Jira. It must be **self-contained** — readable by someone who has no access to the PM's local filesystem.

### Rules

- **Never include local file paths** (e.g., `~/Desktop/...`, `~/Documents/...`, `/Users/...`). They mean nothing to anyone except the PM who ran the pipeline. They are not shareable, not browsable, not useful in a ticket.
- **Never include "Pipeline artifacts:" sections with file path listings.** That is the most common past failure mode of this prompt.
- **Compose each Epic's description from PRD content + the specific scope of that Epic**, not from references to files that produced the PRD.

### What each Epic description should contain

Pull these directly from the PRD content + the specific story scope from the `user_stories.epics[]` entry being processed:

```markdown
**Summary** (3–6 sentences, plain English)
[Use `user_stories.epics[].description` verbatim — this was authored at the user-stories step specifically for stakeholder consumption. Append: "Part of [Feature Name] for [Product]." if not already implied.]

**Stories in this Epic** ([N])
[Bulleted list of story_ids + titles in this Epic, e.g., "US-1.1 — View saved searches", ...]

**Scope (in scope, v1)**
[The subset of PRD Section 2 In Scope items addressed by this Epic's stories. Do not paste the whole PRD scope into every Epic — narrow to this Epic.]

**Out of scope (this Epic)**
[Items that might be expected here but are addressed in a different Epic — e.g., "Saved search notifications: handled in Epic 3, not here."]

**Build sequence within this Epic**
[Local build order — which stories depend on which. Pull from the Sequence Map for this Epic's stories only.]

**Target user(s)**
[From PRD Executive Summary — primary user role + size if known.]

**Success metric(s) this Epic contributes to**
[Concrete numeric targets from PRD that this Epic's stories move the needle on.]

**Owner**
[PM name + team/pod.]

**Phase:** [N from user_stories.epics[].phase]
```

That's a complete, self-contained Epic description. Anyone reading it gets the full picture for *this specific Epic* from Jira alone, without needing to read the whole PRD.

If the breakdown has only one Epic (single-Epic mode), use the original PRD-wide description format — no need to scope to a subset.

### Attach the PRD to each Epic

After creating each Epic, **attach the actual PRD file as an attachment** so anyone who wants the full document can download it from Jira regardless of which Epic they opened.

1. Read the PRD file content from disk:
   `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`
2. Use the Jira MCP's attachment capability (e.g., `mcp__jira__upload-attachment` or the Atlassian-native equivalent) to attach it to each just-created Epic.
3. If attachment fails or the MCP doesn't support it, fall back to: copy the full PRD content into a Jira comment on the Epic instead of attaching as a file. Still no local paths in the description.
4. Also attach the User Stories Breakdown file the same way — it's the source of truth for the tickets.

When multiple Epics are created in one pipeline run, attach to each (the PRD is small text; the redundancy is worth the per-Epic self-containment).

### What NOT to do (anti-patterns to avoid)

- ❌ Listing file paths like `~/Desktop/...prd.md` in the Epic description
- ❌ Writing "See PRD at [path]" instead of including the content
- ❌ Linking to non-shareable local files
- ❌ Leaving the Epic description blank because "the PRD has it"

---

## Connection and fallback

1. **Try Atlassian/Jira MCP first**  
   Call `getAccessibleAtlassianResources` (or equivalent) to confirm the MCP can connect. If the call fails or the MCP is not available:

   - **Do not** create tickets in Jira.
   - **Create a document** at `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-export.md`.
   - The document must include for each PRD user story: **Summary**, **User Story** (As a… I want… So that…), **Acceptance criteria** (Gherkin), **Description** (short context), **Labels** (frontend/backend), and a note that the user can create these in Jira manually or re-run the command when the MCP is connected.
   - Tell the user: "Atlassian MCP is not connected. I created the fallback file with the story content. You can create the tickets in Jira from that file, or run this again when the MCP is available."

   **Fallback document format** (`~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-export.md`):
   - Title: `# Jira stories: <PRD title>`
   - One section per story with: **Summary**, **User Story** (As a… I want… So that…), **Acceptance criteria** (Gherkin), **Description** (short), **Labels** (frontend/backend).
   - Short intro line: "Create these in your team's Jira project (provided at intake) as Stories. Set Epic/parent in Jira when available."

2. **If the MCP connects**  
   Proceed with creating the Epic (if needed) and Stories in Jira. Put each field in the **correct Jira field** (see below). If an Epic cannot be created (e.g. "Field Parent is required") or no parent is provided:

   - **Ask the user:** "I need a parent for the Epic (or for the stories). Please provide an Epic key (e.g. PROJ-123) or a full Jira issue URL."
   - Do not leave stories without a parent without asking; offer the local fallback file in `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/` if they prefer.

---

## Fields to fill for each Story

Map PRD content to these Jira fields. **Keep content in the right place:** User Story and Acceptance criteria go in their dedicated fields, not only in Description.

| Jira field | Source / rule |
|------------|----------------|
| **Summary** | Story title or one-line summary from the PRD (required). |
| **User Story** | The narrative: "As a [role], I want [goal] so that [benefit]." From PRD user stories section. **Must be set in your Jira instance's User Story custom field**, not only in Description. The custom field ID is instance-specific — ask the PM to provide their Jira instance's User Story custom field ID at intake, or look it up via `getJiraIssueTypeMetaWithFields` before creating tickets. |
| **Acceptance criteria** | In **Gherkin format**: `Scenario:`, `Given`, `When`, `Then`, `And` as needed. **Must be set in your Jira instance's Acceptance Criteria custom field**, not only in Description. The custom field ID is instance-specific — ask at intake or look up. |
| **Description** | The **thorough plain-English Description** from the user-stories breakdown's per-story `**Description**` block (3–6 sentences explaining what the ticket is trying to do, written for a stakeholder who hasn't read the PRD). Use it verbatim. Do **not** substitute a terse "PRD Section X" pointer — those references are stale and unhelpful. Do not duplicate the full User Story or all Gherkin here; those go in their dedicated fields. |
| **Parent** | Epic key if known (user provides it or we create an Epic). If Epic creation fails due to required parent, ask the user for Epic key or link. |
| **Team** | The team or pod label provided at intake. If none was provided, leave blank and note it in the result. |
| **Labels** | **frontend** and/or **backend**. Infer from the story: UI, screens, components, copy → **frontend**; API, data, integration, calculations → **backend**. Use both if the story spans both. If unclear, default to one and note it. |
| **Testable** | **Yes** for normal stories (testable by QA). **No** only for non-testable work (e.g. research, docs-only). |
| **Linked work items** | Related issue keys if the PRD or user mentions them. |

---

## Acceptance criteria → Gherkin

Convert each acceptance criterion from the PRD into Gherkin:

- **Scenario:** short label for the behavior.
- **Given:** starting state or context.
- **When:** user or system action.
- **Then:** observable outcome (testable).
- **And:** extend any of the above.

Example:

- PRD: "User can reset password via email link."
- Gherkin:
  ```
  Scenario: User resets password via email link
  Given I am on the login page
  When I request a password reset and click the link in the email
  Then I am taken to a page to set a new password
  And I receive a confirmation after saving
  ```

Keep scenarios testable and specific.

---

## Your process

1. **Get the PRD**  
   User @mentions the PRD file. Read it. Confirm project **AC** and that we're creating **Stories** (and an Epic if needed).

2. **Check Atlassian MCP**  
   Call `getAccessibleAtlassianResources` (or equivalent). If it fails or the tool is unavailable:
   - Create `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-export.md` with one section per PRD user story (Summary, User Story, Acceptance criteria in Gherkin, Description, Labels).
   - Tell the user the MCP is not connected and where the document was created. Stop.

3. **Epics — multi-Epic flow**
   - Read `_pipeline-state.json` → `user_stories.epics[]`. This is the source of truth for how many Epics to create and which stories go under each.
   - For each entry in `user_stories.epics[]`:
     - **Existing-Epic check:** query Jira for `project = [PROJECT] AND issuetype = Epic AND summary ~ "[epic title]"`. If a match exists, surface it to the PM and default to reuse (the PM can override).
     - If no match (or PM chose "create new"): create the Epic with summary = `epic.title`, description composed per the "Epic description" section above, scoped to the stories in this Epic's `story_ids`.
     - Record the resulting Jira Epic key keyed by `epic_id` so step 4 can set the correct `parent` per story.
   - If `user_stories.epics` is missing or empty (pre-v2.4.0 breakdown), fall back to single-Epic mode: create one Epic from the PRD title and use it as `parent` for every story.
   - If creation fails (e.g. "Field Parent is required"): ask the user for an Epic key or Jira link to use as the parent of the new Epic. Do not assume; ask once and use what they provide.

4. **For each user story in the breakdown** (from the user-stories breakdown, `[feature-name]-user-stories.md`):
   - **Summary:** One line (story title following intake title convention if specified).
   - **User Story:** "As a… I want… So that…" from the breakdown (for the **User Story** custom field — see custom field note in step 5).
   - **Acceptance criteria:** Gherkin scenarios verbatim from the breakdown (for the **Acceptance criteria** custom field — see custom field note in step 5).
   - **Description:** Short context only: what the ticket is, PRD section, dependencies, and **"⚠ DRAFT — needs design refresh"** if this story is in `user_stories.draft_stories[]`.
   - **Parent:** Epic key from step 3, looked up by `epic_id` for this story.
   - **Team:** Team/pod label provided at intake (leave blank if none).
   - **Labels:** **frontend** and/or **backend** as appropriate plus any team labels from intake (infer FE/BE from the story; use both if it spans UI and API). **Add `draft` label** if this story appears in `user_stories.draft_stories[]`.
   - **Testable:** Yes (or No only when clearly non-testable).
   - **Linked work items:** Only if the user or PRD specifies related issue keys; also link FE/BE pairs (from the Sequence Map's "Related To" column) as "Relates to".

5. **Create in Jira**  
   Use the Jira MCP to create each Story (and the Epic if you created one). **Set User Story and Acceptance criteria in ADF on create** via `additional_fields` so all fields are populated in one call. If the create API rejects the custom fields, use **editJiraIssue** immediately after create to set them.

   **Custom field IDs are Jira-instance-specific.** Before creating tickets, look up the IDs for your instance using `getJiraIssueTypeMetaWithFields` (or ask the PM to provide them at intake). The IDs shown below are placeholders — replace them with the IDs for the User Story field, Acceptance Criteria field, and Testable? field in the intake-supplied Jira project.

   - **summary:** Story summary.
   - **description:** Short description only (what the ticket is, PRD ref, dependencies). Plain text.
   - **parent:** Epic issue key when applicable.
   - **additional_fields:** Include all of the following when creating each Story:
     - **labels:** e.g. `["frontend", "backend", "<team-label-from-intake>"]`.
     - **customfield_[Testable?]** (Testable?): use the option IDs for Yes / No in your Jira instance.
     - **customfield_[User Story]** (User Story): ADF document. Single paragraph containing the "As a… I want… So that…" text:
       `{"version":1,"type":"doc","content":[{"type":"paragraph","content":[{"type":"text","text":"As a [role], I want [goal] so that [benefit]."}]}]}`
     - **customfield_[Acceptance criteria]** (Acceptance criteria): ADF document. One `paragraph` block per line of Gherkin (each Scenario, Given, When, Then, And as its own paragraph). Example for two scenarios:
       `{"version":1,"type":"doc","content":[{"type":"paragraph","content":[{"type":"text","text":"Scenario: User resets password"}]},{"type":"paragraph","content":[{"type":"text","text":"  Given I am on the login page"}]},{"type":"paragraph","content":[{"type":"text","text":"  When I request a password reset"}]},{"type":"paragraph","content":[{"type":"text","text":"  Then I am taken to set a new password"}]},{"type":"paragraph","content":[{"type":"text","text":"Scenario: Save failure"}]},{"type":"paragraph","content":[{"type":"text","text":"  Given I submitted invalid data"}]},{"type":"paragraph","content":[{"type":"text","text":"  Then I see an error and can retry"}]}]}`
   - If **createJiraIssue** returns an error for the User Story or Acceptance criteria custom field (e.g. "Operation value must be an Atlassian Document"), omit those from `additional_fields` on create and call **editJiraIssue** for that issue with the same ADF payloads in `fields` to set them after create.

6. **Linked work items**  
   If the user or PRD specifies related issues, link them when the API supports it; otherwise note "Link to: [issue key]" in the description or reply.

7. **Confirm**  
   After creating, reply with:  
   - Epic key (if created).  
   - List of created Story keys and summaries.  
   - Reminder to set **Team** in the UI if not set via API (Team field requires an Atlassian Team id; provide Team id if you want it set automatically). **Testable**, **User Story**, and **Acceptance criteria** are set on create (or via edit immediately after).

---

## Important

- **Always use the Jira project the PM provided at intake** and issue type **Story** unless the user says otherwise.
- **If Atlassian MCP is unavailable**, create `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-export.md` with full story content; do not leave the user with nothing. This aligns with the Error Type 4 rule in `ai-framework/error-handling.md`.
- **User Story and Acceptance criteria** go in their dedicated Jira fields in **ADF format**. Set them on create via `additional_fields` using the instance-specific custom field IDs (looked up at intake or via `getJiraIssueTypeMetaWithFields`); if create rejects them, use **editJiraIssue** after create with the same ADF in `fields`.
- **Description** is a short "what this ticket is" (PRD ref, dependencies); avoid duplicate User Story/AC or literal escape characters.
- **If an Epic or parent cannot be created or found**, ask the user for an Epic key or Jira link; do not guess.
- **Labels:** set **frontend** and/or **backend** per story (infer from scope).
- **Gherkin only for Acceptance criteria**; no plain bullets in that field.
- **One Story per PRD user story**; don’t merge multiple PRD stories into one Jira Story.
- If the PRD has no user stories section, extract logical stories from scope/requirements and create them, then note in the description: "Derived from PRD scope; consider updating PRD with these stories."

---

## Ticket Sequencing and Relative Sizing

After all tickets have been created and linked, produce a build sequence plan.

### Step 1 — Dependency mapping

For each ticket, identify:
- Which tickets it depends on.
- Which tickets it blocks.
- Which tickets can be worked in parallel with no dependency.

### Step 2 — Build sequence table

Produce a table with these columns:

| Sequence # | Ticket ID | Ticket Title | Depends On | Can Run Parallel With | Relative Size |
|------------|-----------|--------------|------------|----------------------|---------------|
| 01 | AC-XXX | [title] | none | AC-YYY | S |
| ... | ... | ... | ... | ... | ... |

Relative size:
- **S** — less than a day.
- **M** — one to three days.
- **L** — more than three days or significant unknowns.

### Step 3 — Add to Jira

Add the sequence number as a label in the format `seq-01`, `seq-02`, and so on.
Add the relative size as a label in the format `size-S`, `size-M`, or `size-L`.

Do not modify any other ticket fields.

### Step 4 — Plain English build order summary

Write one paragraph a PM could read aloud in sprint planning. Example shape:

"Start with [tickets]. While that's in progress, [these tickets] can run in parallel. Once the first block is complete, move to [next block]. The final block is [tickets] and depends on [reason]."

### Output

If the breakdown was the input, all of sequence-number, size, depends-on, and parallel-with are already determined. Apply them as labels (`seq-01`, `seq-02`, `size-S`, `size-M`, `size-L`) and as Relates-to / Blocked-by links per the breakdown's Sequence Map. Do not re-derive.

If the fallback was used (no breakdown), produce the table from scratch.

---

## Manifest file (always written)

After all tickets are created (or attempted), write a manifest file at:

```
~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-manifest.md
```

The manifest maps every US-ID from the breakdown to its Jira issue key. Future `/change-mode` runs use this manifest to update the right tickets when a change propagates.

Manifest format:

```markdown
# Jira Manifest — [Feature Name]

**Generated:** [YYYY-MM-DD HH:MM]
**Jira project:** [PROJECT-KEY]
**Epic:** [PROJECT-KEY-N] — [Epic title]

| US-ID | Title | Type | Jira issue key | Jira URL | Status |
|---|---|---|---|---|---|
| US-1.1 | View saved searches | FE | PROJECT-101 | https://... | Created |
| US-1.2 | Saved searches endpoint | BE | PROJECT-102 | https://... | Created |
| US-2.1 | Edit filters | FE | — | — | Failed (write to fallback) |
| ... | ... | ... | ... | ... | ... |

## Summary

- Created: [N]
- Failed: [N]
- Total time: [N] seconds
- Fallback file (if any failures): [path]
```

The manifest must be written every run, even on partial failure.

---

## End-of-run summary (always reported to PM)

After ticket creation completes (success or partial), report to the PM:

```
━━━ Jira Export Complete ━━━

Project: [PROJECT-KEY]
Epic: [PROJECT-KEY-N] — [title] — [URL]

Created: [N] tickets
- FE: [N]
- BE: [N]

Failed: [N] tickets — written to local fallback file:
[path]

Manifest: ~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/jira-export/[feature-name]-jira-manifest.md

Sanity check: opened [PROJECT-KEY] in Jira and confirmed [N] tickets are visible.

Next steps:
- Review the tickets in Jira.
- Run /change-mode if anything needs to be updated across artifacts.
- The manifest will be used for any future updates triggered by /change-mode.
━━━
```

If transient API errors occurred during creation, the prompt should retry ONCE per failed ticket before writing it to the fallback file. Do not retry repeatedly; do not retry permanent errors (e.g., invalid field IDs).
