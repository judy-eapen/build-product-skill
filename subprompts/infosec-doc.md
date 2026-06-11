# InfoSec Questionnaire

Fill out your organization's **InfoSec Questionnaire** (the Ops/DevOps security-review spreadsheet) for a feature, drawing every answer it can from the feature's existing pipeline artifacts, interviewing the PM for the operational facts the artifacts can't supply, and writing the populated `.xlsx` into the feature's `infosec/` folder.

Use when:

- IT / DevOps / the Ops team has asked for a completed InfoSec questionnaire before a feature can be deployed or reviewed.
- A feature has at least a PRD (ideally also a system design / technical review) and the PM wants the security doc pre-filled instead of typing every cell by hand.

**Standalone command. Does _not_ run automatically inside `/build-product`** — the PM invokes it once the feature has enough artifacts to answer from (PRD minimum; system design + technical review strongly recommended).

---

## The cardinal rule — never fabricate a security answer

This document goes to a security review. A wrong-but-confident answer is worse than a blank one.

- Fill a cell **only** when the answer is either (a) directly stated or unambiguously implied by a pipeline artifact, or (b) confirmed by the PM in the interview step.
- For a value that is a defensible industry-standard default but **not** confirmed by an artifact (e.g. "TLS 1.2+ in transit"), fill it **and** list it in the in-thread report under "Derived defaults — confirm before sending" so the PM can verify.
- Anything you cannot source and the PM does not answer → write the literal string **`⚠ NEEDS INPUT`** in the cell. Never guess vendor names, contacts, SLAs, encryption details, severity, or legal-review status.

---

## Inputs (Step 0 — resolve feature, artifacts, and template)

1. **Feature.** Auto-derive from the working folder / conversation. If ambiguous, ask which feature (the folder name under the feature workspace root).
2. **Required artifact:** the PRD at `~/Desktop/Resources/PDLC Workflow Docs/[feature]/prd/[feature]-prd.md`. If there is no PRD, refuse and direct the PM to `/create-prd` first.
3. **Recommended artifacts** (read if present, skip gracefully if not — each missing one just means more cells fall to the interview):
   - `technical-review/[feature]-system-design.md` — hosting, data structures, APIs, auth, integrations, tech stack. **Richest source for infrastructure and security tabs.**
   - `technical-review/[feature]-technical-review.md` — CTO-review risks, NFRs, security concerns.
   - `codebase-review/[feature]-codebase-review.md` — existing standards alignment, service accounts, integrations.
   - `diagrams/[feature]-feature-diagram.md` and `_pipeline-state.json` → `export_urls` — the technical diagram link for documentation cells, plus owner / Confluence / Jira links.
   - `timeline/` and `_pipeline-state.json` → `intake` — planning / implementation dates, PM and team.
4. **Template.** Ask the PM:

   > "Please provide the path to your organization's InfoSec questionnaire template (`.xlsx` file). This is typically a read-only master maintained by your Ops/DevOps/Security team.
   > If you're not sure where it is, check your shared drive, your Ops team's documentation, or ask your DevOps contact."

   If it is not found at the path provided, ask again before proceeding. The template is read-only — **never write to the master.** Copy it per feature (Step 3).

Read every available artifact fully before drafting answers. Use `openpyxl` to introspect the template at runtime — do not assume a fixed structure (see Step 1).

---

## Step 1 — Introspect the template

Before deriving any answers, load the template to discover its structure:

```python
import openpyxl
wb = openpyxl.load_workbook(template_path, read_only=True)
print("Sheets:", wb.sheetnames)
for name in wb.sheetnames:
    ws = wb[name]
    print(f"\n=== {name} ===")
    for row in ws.iter_rows(min_row=1, max_row=40, values_only=True):
        if any(cell is not None for cell in row):
            print(row)
```

From the output, identify:
- **Sheet names** — the tabs to fill.
- **Column layout** — which column contains questions, which contains hints/descriptions, and which is the answer column. (Often col A = question, col B = hint, col C = answer — but derive from the actual headers, not assumptions.)
- **Answer cell references** — the specific cells where answers belong, per tab.
- **Dropdown/validation fields** — cells with data-validation constraints (e.g., a severity dropdown). Note the valid values; write only values the dropdown accepts.
- **Merged cells** — for any merged range in the answer column, always write to the top-left cell of the merge.
- **Read-only fields** — cells labeled for Ops/reviewer use only (e.g., "reviewed by", "accepted date"). Leave these blank.

Report the discovered structure to the PM:
```
Template: [path]
Sheets:   [tab1, tab2, ...]
Answer column: [C or as discovered]
Dropdown cells: [list with valid values]
Read-only cells: [list]
```

If the template structure is unclear or non-standard, ask the PM to clarify before proceeding.

---

## Step 2 — Derive answers from the artifacts

Work through each sheet and each question row in the template. For each question, read the question text in the question column, then classify and derive the answer.

**Provenance tags (used in the Step 5 report):**
- `[artifact]` — directly answerable from PRD, system design, technical review, codebase review, diagrams, or state
- `[default]` — industry-standard default that should be filled but flagged for PM confirmation
- `[interview]` — operational fact that only the PM or team can supply
- `[ops]` — leave blank; Ops/reviewer fills it
- `[n/a]` — not applicable for this feature type

**Common question-to-artifact mappings** (adapt to the actual question text in your template):

| Question theme | Source | Notes |
|---|---|---|
| Product/service name | PRD title or feature name | `[artifact]` |
| Description / purpose | PRD §1 problem statement | `[artifact]` |
| Vendor / built-by | PRD: internally built vs. third-party SaaS | `[artifact]` |
| Primary stakeholder / sponsor | `_pipeline-state.json` → `intake` / PRD masthead | `[artifact]` / `[interview]` |
| Primary tech lead | PRD / system design / state | `[artifact]` / `[interview]` |
| Team responsible | `intake` / PRD | `[artifact]` |
| Planning / implementation dates | `intake` / timeline | `[artifact]` / `[interview]` |
| Internal vs. external product | PRD product type | `[artifact]` |
| Technical documentation link | Figma diagram URL (`export_urls.figma_diagram_url`) + system-design path | `[artifact]` |
| Architecture / tech review status | Technical review if present, else `[interview]` | |
| Product environments | System design hosting section | `[artifact]` / `[default]` |
| PII / sensitive data | PRD / system-design data model | `[artifact]` |
| Data hosted externally? | System design hosting/integrations | `[artifact]` |
| Data outside home region? | System design hosting region | `[artifact]` / `[interview]` |
| Service accounts | System design integrations / codebase review | `[artifact]` / `[interview]` |
| Encryption at rest | System design (if specified), else `[default]` (AES-256) | Confirm with PM |
| Encryption in transit | System design (if specified), else `[default]` (TLS 1.2+) | Confirm with PM |
| Vendor/default credentials changed? | `[interview]` | |
| Per-environment unique credentials? | `[interview]` / codebase review | |
| Messaging / notifications? | PRD features | `[artifact]` |
| Logging / observability | System design observability section | `[artifact]` / `[interview]` |
| Infrastructure tagging? | Codebase review / `[interview]` | |
| Hostname conventions? | Codebase review / `[interview]` | |
| CI/CD alignment? | Codebase review | `[artifact]` / `[interview]` |
| Code standards deviation? | Codebase review | `[artifact]` / `[interview]` |
| PR approvals enforced? | Codebase review (branch protection) | `[artifact]` / `[interview]` |
| Static analysis (SonarCloud, Snyk, etc.)? | Codebase review | `[artifact]` / `[interview]` |
| Product runbook link | Workspace if present, else `[interview]` | |
| Authentication method | System design auth section | `[artifact]` / `[interview]` |
| Authorization / permissions model | System design auth section | `[artifact]` / `[interview]` |
| Security response team | Owning team from `intake` | `[interview]` |
| Infrastructure patching owner | `[interview]` | |
| Will this use AI? | PRD product type / system design | `[artifact]` |
| AI service name(s) | System design | `[artifact]` / `[interview]` |
| AI services on org's approved list? | `[interview]` — ask for the org's current approved list | |
| AI data-sharing opted out? | `[interview]` | |
| AI service SLA | `[interview]` / vendor documentation | |
| Product severity / criticality | **Always `[interview]`** — drives DR requirements; propose from PRD NFRs but confirm | |
| Escalation contacts (normal / critical) | `[interview]` | |
| Emergency contact | `[interview]` | |
| Disaster recovery environment | System design (if covered), else `[interview]` | |
| DR in separate geographic region? | System design (regions), else `[interview]` | **Flag if severity is Critical/High and DR is blank** |
| Vendor support hours | `[interview]` for third-party; "internally supported" for internal builds | |
| Vendor SLA | `[interview]` / vendor documentation | |
| Legal sign-off / MSA/NDA | `[interview]` | |
| Reviewed / accepted by | Leave blank — `[ops]` | |

For questions not in this table, use the question text and any hint text to infer the best source. When in doubt, classify as `[interview]` — it is always better to ask than to fabricate.

---

## Step 3 — Interview for the gaps

After deriving everything you can from artifacts, collect **every** `[interview]` item and every `[default]` you want confirmed, and ask the PM in **one batched `AskUserQuestion` round** (group logically: e.g., Service & Support, Data & Access, Standards, AI). Pre-fill each with your best proposal so the PM can accept-or-edit rather than type from scratch.

- **Severity / criticality is always asked** — it gates the DR requirement. Propose based on PRD criticality/NFRs.
- The PM may answer "unknown" / "skip" for any item → that cell becomes `⚠ NEEDS INPUT`.
- Do not loop more than necessary; one good batched round, then fill.

---

## Step 4 — Write the `.xlsx`

1. Ensure `openpyxl` is available (`python3 -c "import openpyxl"`; `pip install openpyxl` if not).
2. **Introspect, don't trust the question-mapping blindly.** Before writing, re-confirm each answer cell by matching the question text in the question column. The layout described in Step 1 reflects the live template — if rows shifted or a tab was renamed, re-derive from the actual headers. Write to the top-left cell of any merged range.
3. **Preserve the template.** Copy the master to the output path each run (this re-copies from the golden original so stale answers never linger). Load the copy with `openpyxl.load_workbook(out_path)` — preserving styles, dropdown data-validation, and formatting. Set `.value` on answer cells only; touch nothing else. Leave reviewer/Ops-only fields blank.
4. Save to `~/Desktop/Resources/PDLC Workflow Docs/[feature]/infosec/[feature]-infosec-questionnaire.xlsx`. Create the `infosec/` folder if needed. **Idempotent** — re-running overwrites this file.

Reference fill script (adapt the `answers` dict from Steps 2–3):

```python
import openpyxl, shutil, os

TEMPLATE = "<path provided by PM at Step 0>"
feature  = "<feature-name>"
out_dir  = os.path.expanduser(f"~/Desktop/Resources/PDLC Workflow Docs/{feature}/infosec")
out_path = os.path.join(out_dir, f"{feature}-infosec-questionnaire.xlsx")
os.makedirs(out_dir, exist_ok=True)
shutil.copyfile(TEMPLATE, out_path)          # start from a clean copy of the golden template
wb = openpyxl.load_workbook(out_path)

# answers: { sheet_name: { cell_ref: value } } — only answer cells, built from Steps 2–3
# Sheet names and cell refs come from Step 1 introspection, not hardcoded assumptions
answers = {
    "Sheet 1 name": {"C14": "...", ...},
    "Sheet 2 name": {"C6": "...", ...},
    # ... one entry per sheet
}
for sheet, cells in answers.items():
    ws = wb[sheet]
    for cell_ref, val in cells.items():
        ws[cell_ref] = val
wb.save(out_path)
print("wrote", out_path)
```

---

## Step 5 — Report & record

Print an in-thread summary (the xlsx is the only file, so this is the review surface):

- **Filled from artifacts** — count + the notable ones, each with its source doc.
- **Derived defaults — confirm before sending** — every `[default]` cell, so the PM verifies.
- **From your interview answers** — what the PM supplied.
- **⚠ Still NEEDS INPUT** — every unresolved cell, grouped by tab, so the PM knows exactly what to finish before sending to Ops.
- If severity is Critical/High and disaster recovery fields are blank or "No", call it out as a likely review blocker.

Then record the artifact in `_pipeline-state.json` if the feature has one: add `export_urls.infosec_doc` (the file path) and a top-level note. Don't fail if state is absent.

Finally, point the PM at the file and remind them: review the defaults, complete any `⚠ NEEDS INPUT` cells, get any required legal documents (MSA, NDA, etc.) reviewed per your organization's policy, then send to the Ops/DevOps/Security team — and that any later change to the technical diagram must be re-communicated to the reviewer for re-review.
