# Lint Style

Check a generated document against `ai-framework/style-preferences.md` and flag violations before the document is finalized. Run this as a post-generation check on any PRD, review, user-stories breakdown, or design catalog.

---

## When to run

- Automatically: after generating any document longer than 200 words, before writing it to disk.
- Manually: `/lint-style [file path]` — run against an existing file.

---

## Execution

### Step 1 — Load style rules

Read `ai-framework/style-preferences.md`. If it is empty, print "No style rules configured — skipping lint." and stop.

Extract each rule as a checkable pattern. Map each rule to a detection method:

| Rule | Detection |
|------|-----------|
| No em dashes | grep for `—` (U+2014) |
| Bullet points over paragraphs | flag prose blocks longer than 4 sentences with no bullets |
| No filler phrases | grep for: "great question", "happy to help", "absolutely,", "just ", "very ", "certainly", "of course", "I'd be happy" (case-insensitive) |
| Direct tone | flag hedging phrases: "might be", "could potentially", "it may be worth", "you might want to consider" |
| State assumptions explicitly | no automated check — manual reviewer note only |
| No invented facts | no automated check — flag as "verify" when specific API names, model names, or version numbers appear |

For rules without automated checks, note them as "manual review required."

### Step 2 — Scan the document

Scan the document content against every detectable rule. Record:
- Rule violated
- Line or approximate location
- Offending text (first 60 characters of the match)
- Suggested fix (one line)

### Step 3 — Report

If no violations are found:
```
Style lint passed. No violations found.
```

If violations are found:
```
━━━ Style Lint Results ━━━

[N] violations found:

1. [Rule] — Line ~[N]
   Found: "[offending text snippet]"
   Fix: [one-line suggestion]

2. [Rule] — Line ~[N]
   Found: "[offending text snippet]"
   Fix: [one-line suggestion]

Manual review notes:
- Verify any specific API names, model names, or version numbers are accurate.
- Check that all assumptions are stated explicitly, not implied.
━━━
```

### Step 4 — Fix or flag

If run as an automatic post-generation check (before writing to disk):
- Fix violations that have a clear mechanical fix (e.g., replace `—` with ` - ` or rewrite as a bullet).
- For violations that require judgment (hedging, filler phrases), flag them and ask the PM: "Fix these before saving, or save as-is and fix manually?"

If run as a standalone `/lint-style` against an existing file:
- Report violations only. Do not auto-fix existing files without PM confirmation.
- Ask: "Fix these in place? (yes / no)"

---

## Integration points

The following steps run lint automatically before writing output to disk:
- Step 2 — Create PRD (lint the full PRD before saving)
- Step 3a — Product Review (lint the review before saving)
- Step 3b — Technical Review (lint the review before saving)
- Step 10 — User Stories Breakdown (lint the full breakdown before saving)

All other steps: lint is optional. Run manually with `/lint-style [path]` if needed.
