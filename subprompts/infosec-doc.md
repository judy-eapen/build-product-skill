# InfoSec Questionnaire

Fill out Bright's **InfoSec Questionnaire** (the Ops/DevOps security-review spreadsheet) for a feature, drawing every answer it can from the feature's existing pipeline artifacts, interviewing the PM for the operational facts the artifacts can't supply, and writing the populated `.xlsx` into the feature's `infosec/` folder.

Use when:

- IT / DevOps / the Ops team has asked for a completed InfoSec questionnaire before a feature can be deployed or reviewed.
- A feature has at least a PRD (ideally also a system design / technical review) and the PM wants the security doc pre-filled instead of typing every cell by hand.

**Standalone command. Does _not_ run automatically inside `/build-product`** — the PM invokes it once the feature has enough artifacts to answer from (PRD minimum; system design + technical review strongly recommended).

---

## The cardinal rule — never fabricate a security answer

This document goes to a security review. A wrong-but-confident answer is worse than a blank one.

- Fill a cell **only** when the answer is either (a) directly stated or unambiguously implied by a pipeline artifact, or (b) confirmed by the PM in the interview step.
- For a value that is a defensible Bright-standard default but **not** confirmed by an artifact (e.g. "TLS 1.2+ in transit"), fill it **and** list it in the in-thread report under "Derived defaults — confirm before sending" so the PM can verify.
- Anything you cannot source and the PM does not answer → write the literal string **`⚠ NEEDS INPUT`** in the cell. Never guess vendor names, contacts, SLAs, encryption details, severity, or legal-review status.

---

## Inputs (Step 0 — resolve feature, artifacts, and template)

1. **Feature.** Auto-derive from the working folder / conversation. If ambiguous, ask which feature (the folder name under `~/Desktop/Resources/PDLC Workflow Docs/`).
2. **Required artifact:** the PRD at `~/Desktop/Resources/PDLC Workflow Docs/[feature]/prd/[feature]-prd.md`. If there is no PRD, refuse and direct the PM to `/create-prd` first.
3. **Recommended artifacts** (read if present, skip gracefully if not — each missing one just means more cells fall to the interview):
   - `technical-review/[feature]-system-design.md` — hosting, data structures, APIs, auth, integrations, tech stack. **Richest source for tabs 3–7.**
   - `technical-review/[feature]-technical-review.md` — CTO-review risks, NFRs, security concerns.
   - `codebase-review/[feature]-codebase-review.md` — existing standards alignment, service accounts, integrations, SonarCloud/Snyk usage.
   - `diagrams/[feature]-feature-diagram.md` and `_pipeline-state.json` → `export_urls` — the technical diagram link (Figma) for the "Technical Documentation" cell, plus owner / Confluence / Jira links.
   - `timeline/` and `_pipeline-state.json` → `intake` — planning / implementation dates, PM and team.
4. **Golden template.** The master is read-only at:
   `~/Desktop/Resources/AI-Automation/documents /InfoSec Questionnaire Ops Team 2026 -Template.xlsx`
   (note the trailing space in the `documents ` folder name). Never write to it. Copy it per feature. If it is not found, ask the PM for the current template path (Ops may have reissued it) before proceeding.

Read every available artifact fully before drafting answers. Use `openpyxl` to introspect the template at runtime (do not trust the cell map below blindly — see Step 3).

---

## Step 1 — Derive answers from the artifacts

The questionnaire has **7 tabs**, all laid out as `Question (col A) | Hint (col B) | Answer (col C)`. Work tab by tab. For each cell, the table below gives the **primary source** and **how to derive it**. Mark provenance for the report: `[artifact]` = pulled from a doc, `[default]` = Bright-standard default needing confirmation, `[interview]` = must ask, `[ops]` = Ops fills it, leave blank.

### Tab 1 — Product/Service Overview
| Cell | Question | Source & derivation |
|------|----------|---------------------|
| C14 | Product/Service Name | Feature name / PRD title. `[artifact]` |
| C15 | Vendor name | "Bright" for an internally-built feature. If the PRD/system design names a third-party SaaS or vendor, use it. `[artifact]` |
| C16 | Description of product/service | One-sentence purpose from PRD §1 / problem statement. `[artifact]` |
| C17 | Primary Bright Stakeholder | ELT sponsor from `_pipeline-state.json` → `intake` / PRD masthead. Often `[interview]`. |
| C18 | Primary Bright Tech Lead | From PRD / system design / state. Often `[interview]`. |
| C19 | Team Responsible | From `intake` / PRD (e.g. "AC pod"). `[artifact]` |
| C20 | Planning Start Date | From `intake` / timeline. `[artifact]` else `[interview]`. |
| C21 | Implementation Start Date | From timeline. `[artifact]` else `[interview]`. |
| C22 | Bright-only or SaaS? | Internal tool → "Bright only"; externally-distributed product → "SaaS". From PRD product type. `[artifact]` |

### Tab 2 — Service & Support Details
| Cell | Question | Source & derivation |
|------|----------|---------------------|
| C6 | Product Severity? | **Dropdown** (Critical/High/Medium/Low). Derive a *proposal* from PRD criticality/NFRs but **confirm in interview** — this drives DR requirements. `[interview]` |
| C8 | Escalation – Normal | Runbook / technical review if present, else `[interview]`. |
| C9 | Escalation – Critical | Same. `[interview]` |
| C10 | Bright Emergency Contact | `[interview]` |
| C11 | Vendor Support | "N/A — Bright-built/-supported" for internal builds; vendor hours if third-party. `[interview]` |
| C12 | Disaster Recovery env. | From system design hosting section if covered, else `[interview]`. |
| C13 | DR in separate geographic locations? | From system design (regions) else `[interview]`. **DR is mandatory for Critical/High severity — flag if severity is Critical/High and this is blank.** |

### Tab 3 — Hosting & Architecture
| Cell | Question | Source & derivation |
|------|----------|---------------------|
| C5 | Product residence? | "Bright" / "AWS" / 3rd-party — from system design hosting/tech stack. `[artifact]` |
| C6 | Internal/External Product? | Internal tool → "Internal"; consumer/agent-facing → "External"; both → "Both". From PRD. `[artifact]` |
| C7 | Technical Documentation | Cite the Figma diagram URL (`export_urls.figma_diagram_url`) **and** the system-design doc path. `[artifact]` |
| C8 | Architectural Team Review? | "Yes – [date]" only if an artifact says so; otherwise "Pending" / `[interview]`. |
| C9 | Product environments? | "PRD / TST / DEV" or as stated in system design. `[artifact]` else `[default]`. |
| C10 | Environmental deviation? | From system design; "None" only if confirmed, else `[interview]`. |

### Tab 4 — Data Protection & Access
| Cell | Question | Source & derivation |
|------|----------|---------------------|
| C4 | PII shared externally? | From PRD/system-design data model. State which fields if yes. `[artifact]` |
| C5 | Sensitive data? | PII/credentials/secrets in the data structures — list and say where stored/secured. `[artifact]` |
| C6 | Impact – Data compromise? | Reason from data sensitivity (CIA impact). `[artifact]` |
| C7 | Bright data off-site hosted? | "No" if Bright/AWS-hosted; "Yes – details" if vendor. `[artifact]` |
| C8 | Bright data outside US? | "No" if us-east/us-west only. `[artifact]` else `[interview]`. |
| C9 | Product service accounts? | From system design integrations / codebase review (e.g. "read-only Matrix DB account"). `[artifact]` else `[interview]`. |
| C10 | Encryption – rest? | "AES-256 with AWS-managed keys" is the Bright default — `[default]`, confirm. If system design specifies, `[artifact]`. |
| C11 | Encryption – transit? | "TLS 1.2+" Bright default — `[default]`, confirm. `[artifact]` if stated. |
| C12 | Vendor default credentials changed? | `[interview]` (or "N/A — Bright-built"). |
| C13 | Environmentally unique credentials? | `[default]` "Yes" for Bright builds, confirm. |
| C14 | Vendor access required? | `[interview]` (or "No" for Bright-built). |
| C15 | Messaging functionality? | From PRD — if the feature sends email/Slack/SMS/push (e.g. notifications), say which. `[artifact]` |

### Tab 5 — Platform & Engineering Standards
For a Bright-built feature most of these are "Yes — aligned"; only assert it when codebase review / technical review supports it, else `[default]` (confirm) or `[interview]`.
| Cell | Question | Source & derivation |
|------|----------|---------------------|
| C5 | Alignment – tagging? | `[default]` "Yes" / from codebase review. |
| C6 | Alignment – logging & responsibility (SumoLogic)? | From system design observability section, else `[default]`/`[interview]`. |
| C7 | Alignment – hostnames? | `[default]`/codebase review. |
| C8 | Alignment – coding (any deviation)? | From codebase review. `[artifact]` else `[default]`. |
| C9 | Alignment – Terraform? | `[default]`/`[interview]`. |
| C10 | PR Approvals Enforced? | From codebase review (branch protection) else `[interview]`. |
| C11 | Alignment – CI/CD? | `[default]`/codebase review. |
| C12 | Integration monitoring? | From system design; list API/customer-facing endpoints to monitor. `[artifact]` else `[interview]`. |
| C13 | Who gets notified of alerts? | `[interview]`. |
| C14 | Any deviations from Bright standards? | From technical/codebase review; "None known" only if confirmed. `[artifact]` else `[interview]`. |
| C15 | Integration – SonarCloud? | From codebase review else `[interview]`. |
| C16 | Integration – Snyk? | From codebase review else `[interview]`. |
| C17 | Product Runbook? (link) | Link if one exists in the workspace, else `[interview]`. |

### Tab 6 — Security & Maintenance
| Cell | Question | Source & derivation |
|------|----------|---------------------|
| C4 | Product authentication? | From system design auth section (e.g. "Okta SSO"). `[artifact]` else `[interview]`. |
| C5 | Team responsible for ongoing security response? | Owning pod from `intake` (e.g. "AC pod") — confirm. `[default]`/`[interview]`. |
| C6 | Product authorization? | From system design (e.g. "Bright Permissions / role-based"). `[artifact]` else `[interview]`. |
| C7 | Infrastructure update management? | Patching owner/process — `[interview]` unless stated. |

### Tab 7 — AI Usage
| Cell | Question | Source & derivation |
|------|----------|---------------------|
| C6 | Will this service utilize AI? | "Yes"/"No" from PRD product type. `[artifact]` |
| C7 | What AI service(s)? | From system design (e.g. "Claude via AWS Bedrock", "OpenAI"). `[artifact]` else `[interview]`. |
| C8 | All AI services on Bright's approved list? | `[interview]` ("reach out to Ops for the latest list"). |
| C9 | Opted out of data sharing for all AI services? | `[interview]`. |
| C10 | AI service SLA? | `[interview]` / vendor SLA. |

If C6 is "No", set C7–C10 to "N/A".

---

## Step 2 — Interview for the gaps

After deriving everything you can, collect **every** `[interview]` cell and every `[default]` you want confirmed, and ask the PM in **one batched `AskUserQuestion` round** (group logically: Service & Support, Data/Access, Standards, AI). Pre-fill each with your best proposal so the PM can accept-or-edit rather than type from scratch.

- Severity (C6, tab 2) is always asked — it gates the DR requirement.
- The PM may answer "unknown" / "skip" for any item → that cell becomes `⚠ NEEDS INPUT`.
- Do not loop more than necessary; one good batched round, then fill.

---

## Step 3 — Write the `.xlsx`

1. Ensure `openpyxl` is available (`python3 -c "import openpyxl"`; `pip install openpyxl` if not).
2. **Introspect, don't trust blindly.** Load the template; for each sheet confirm the Answer column by finding the header cell whose value is `Answer`, and confirm each question row by matching the question text in column A. The cell map above reflects the 2026 template — if the live template's structure differs, re-derive from the headers and report the discrepancy. **Write to the top-left cell of any merged range** (the severity answer is merged `C6:C7` → write `C6`).
3. Preserve the template — load and save the workbook itself (this keeps styles, the severity dropdown's data-validation, and all formatting). Only set `.value` on answer cells; touch nothing else. Leave `E1`/`E2` (Review-and-accepted-by / Date) blank — Ops fills those.
4. Save to **`~/Desktop/Resources/PDLC Workflow Docs/[feature]/infosec/[feature]-infosec-questionnaire.xlsx`**. Create the `infosec/` folder if needed. **Idempotent** — re-running overwrites this file (re-copy from the golden template each run so stale answers never linger).

Reference fill script (adapt the `answers` dict from Steps 1–2):

```python
import openpyxl, shutil, os
TEMPLATE = "/Users/judydarvin/Desktop/Resources/AI-Automation/documents /InfoSec Questionnaire Ops Team 2026 -Template.xlsx"
OUT_DIR  = os.path.expanduser("~/Desktop/Resources/PDLC Workflow Docs/[feature]/infosec")
OUT      = os.path.join(OUT_DIR, "[feature]-infosec-questionnaire.xlsx")
os.makedirs(OUT_DIR, exist_ok=True)
shutil.copyfile(TEMPLATE, OUT)                           # start from a clean copy of the golden template
wb = openpyxl.load_workbook(OUT)

# answers: { sheet_name: { cell: value } } — only answer cells, built from Steps 1–2
answers = {
  "Product Service Overview": {"C14": "...", "C15": "Bright", ...},
  "Service & Support Details": {"C6": "Medium", "C8": "...", ...},
  "Hosting & Architecture": {...},
  "Data Protection & Access": {...},
  "Platform & Engineering Standard": {...},
  "Security & Maintenance": {...},
  "AI Usage": {...},
}
for sheet, cells in answers.items():
    ws = wb[sheet]
    for cell, val in cells.items():
        ws[cell] = val
wb.save(OUT)
print("wrote", OUT)
```

(Sheet names exactly: `Product Service Overview`, `Service & Support Details`, `Hosting & Architecture`, `Data Protection & Access`, `Platform & Engineering Standard`, `Security & Maintenance`, `AI Usage`.)

---

## Step 4 — Report & record

Print an in-thread summary (the xlsx is the only file, so this is the review surface):

- **Filled from artifacts** — count + the notable ones, each with its source doc.
- **Derived defaults — confirm before sending** — every `[default]` cell, so the PM verifies.
- **From your interview answers** — what the PM supplied.
- **⚠ Still NEEDS INPUT** — every unresolved cell, grouped by tab, so the PM knows exactly what to finish before sending to Ops.
- If severity is Critical/High and DR (C12/C13) is blank or "No", call it out as a likely review blocker.

Then record the artifact in `_pipeline-state.json` if the feature has one: add `export_urls.infosec_doc` (the file path) and a top-level note that the InfoSec questionnaire was generated. Don't fail if state is absent.

Finally, point the PM at the file and remind them: review the defaults, complete any `⚠ NEEDS INPUT` cells, get the required legal docs (MSA/NDA) reviewed by Legal per the template's disclaimer, then send to the Ops team — and that any later change to the technical diagram must be re-communicated to Ops for re-review.
