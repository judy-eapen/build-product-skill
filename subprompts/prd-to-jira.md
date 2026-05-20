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

## Knowledge Base Sizing Check

Before creating tickets, check whether `~/Desktop/Resources/PDLC Workflow Docs/_knowledge-base.md` exists.

- If it does, read the sizing accuracy entries from past features. Look for a consistent pattern of undersizing or oversizing a particular type of work.
- If a pattern exists, surface it to the PM as a warning. Ask whether to adjust the size labels from the breakdown up or down based on the pattern. (The breakdown's sizes are starting points; the knowledge base may suggest adjustments.)
- If no pattern exists or the knowledge base does not exist, skip silently.

---

## Pre-flight: validate custom field IDs

Custom field IDs are Jira-instance-specific. Before bulk creation, validate them. Fail fast if wrong.

1. Use `getJiraIssueTypeMetaWithFields` (or equivalent) to list the available custom fields for the intake-supplied Jira project + Story issue type.
2. Identify the field IDs for: User Story narrative, Acceptance Criteria, Testable (yes/no), and any team / role fields the PM specified at intake.
3. If any required custom field cannot be located, ask the PM to provide the field ID before proceeding. Do not guess.
4. Test the IDs against ONE field on ONE issue (e.g., create a dry-run draft, then delete) before bulk creation if the MCP allows.

If validation fails, write the entire intended export to the local fallback file (Error Type 4) and notify the PM. Do not create partial tickets.

---

Create Jira Stories (and optionally an Epic) from the approved user-stories breakdown. Create tickets in the **Jira project provided at intake** (see Intake Parameters in `CLAUDE.md`). Fill the fields the PM uses when creating tickets manually.

## When to use

- Step 11 of the Work pipeline: input is the approved breakdown (Gate 3 already passed).
- Standalone: a PM has a PRD and wants Jira tickets directly. In this case, infer from the PRD's Phased Plan.
- Default: create tickets in the **Jira project name** the PM provided at intake, issue type **Story**. If the PM did not provide a project name, ask before creating any tickets.
- If the user provides an existing **Epic** key or link, use it as Parent for all stories. Otherwise, create one Epic from the PRD title (if a parent is available), then create Stories under it. See the next section for what goes in the Epic description.

---

## Epic description — what to write (and what NOT to write)

The Epic description is what every stakeholder sees first when they open the Epic in Jira. It must be **self-contained** — readable by someone who has no access to the PM's local filesystem.

### Rules

- **Never include local file paths** (e.g., `~/Desktop/...`, `~/Documents/...`, `/Users/...`). They mean nothing to anyone except the PM who ran the pipeline. They are not shareable, not browsable, not useful in a ticket.
- **Never include "Pipeline artifacts:" sections with file path listings.** That is the most common past failure mode of this prompt.
- **Compose the description from PRD content**, not from references to files that produced the PRD.

### What the Epic description should contain

Pull these directly from the PRD content (not from file paths):

```markdown
**Summary** (1–2 sentences, plain English)
v1 of the [Feature Name] for [Product]. [One-sentence what it does.]

**Scope (in scope, v1)**
[The In Scope bullet list from PRD Section 2, written as a paragraph or short bullets — not just "see PRD".]

**Out of scope (v1)**
[The Out of Scope bullet list from PRD Section 2, same treatment.]

**Build sequence**
Phase 0: [N] verification tickets. Phase 1: [N] build tickets across [N] waves.
[Optionally: one-paragraph build-order summary copied verbatim from the User Stories Breakdown's Sequence Map summary.]

**Target user(s)**
[From PRD Executive Summary — primary user role + size if known, e.g. "Logged-in [Product] users (consumers + agents), ~4,000 MAU".]

**Success metric(s)**
[From PRD Executive Summary success metrics — concrete numeric targets only.]

**Owner**
[PM name + team/pod, e.g., "Jane Doe (PM, Acme pod)".]
```

That's a complete, self-contained Epic description. Anyone reading it gets the full picture from Jira alone.

### Attach the PRD to the Epic

After creating the Epic, **attach the actual PRD file as an attachment** to the Epic so anyone who wants the full document can download it from Jira.

1. Read the PRD file content from disk:
   `~/Desktop/Resources/PDLC Workflow Docs/[feature-name]/prd/[feature-name]-prd.md`
2. Use the Jira MCP's attachment capability (e.g., `mcp__jira__upload-attachment` or the Atlassian-native equivalent) to attach it to the just-created Epic.
3. If attachment fails or the MCP doesn't support it, fall back to: copy the full PRD content into a Jira comment on the Epic instead of attaching as a file. Still no local paths in the description.
4. Optional: also attach the User Stories Breakdown file the same way — it's the source of truth for the tickets.

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
| **Description** | A **short description of what the ticket is**: PRD section reference, dependencies, and context. Do not duplicate the full User Story or all Gherkin here; keep it concise (e.g. "PRD Section 6 (US-1). Depends on stats endpoint."). |
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

3. **Epic / Parent**  
   - If the user gives an **existing Epic key or link** (e.g. PROJ-1200 or any Jira issue URL): extract the key and use it as Parent for all stories.  
   - If not: try to create one **Epic** in the intake-supplied Jira project with summary = PRD title and description = PRD link/summary. If creation fails (e.g. "Field Parent is required"): **ask the user** for an Epic key or Jira link to use as parent for the new Epic or for the stories. Do not assume; ask once and use what they provide.
   - Optional clarify: "Use Epic [key] as parent, or should I create a new Epic from the PRD?"

4. **For each user story in the PRD** (from "User stories & requirements" or equivalent):
   - **Summary:** One line (story title).
   - **User Story:** "As a… I want… So that…" from the PRD (for the **User Story** custom field — see custom field note in step 5).
   - **Acceptance criteria:** Gherkin scenarios (for the **Acceptance criteria** custom field — see custom field note in step 5).
   - **Description:** Short context only: what the ticket is, PRD section, dependencies (e.g. "PRD Section 6 (US-1). Depends on stats endpoint.").
   - **Parent:** Epic key from step 3.
   - **Team:** Team/pod label provided at intake (leave blank if none).
   - **Labels:** **frontend** and/or **backend** as appropriate plus any team labels from intake (infer FE/BE from the story; use both if it spans UI and API).
   - **Testable:** Yes (or No only when clearly non-testable).
   - **Linked work items:** Only if the user or PRD specifies related issue keys.

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
