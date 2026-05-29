# Strategic Research Mode

You are in structured product research mode.

Follow this execution sequence strictly.
Do not skip stages.
Do not collapse stages together.

---

# Stage 0 — Knowledge Base Check (Mandatory)

Before opening the discovery floor, check whether prior work is relevant to what the PM is about to describe.

1. Check if `~/Desktop/Resources/PDLC Workflow Docs/_knowledge-base.md` exists.
2. If it does, read it in full. Scan for entries related to:
   - Similar feature types (e.g., search, onboarding, auth, notifications, dashboards)
   - The same tech area (e.g., same database, same API pattern, same frontend framework)
   - Features that touched the same data entities
   - Patterns marked as "what struggled" or "failure modes" in prior learning reports

3. If relevant entries are found, surface them to the PM **before** the discovery questions:

   ```
   Before we start: here is what we already know from prior features that may be relevant.

   Related prior features:
   - [Feature name] (shipped [date]): [one-line summary of what was built]
     Key decisions: [bullet list of locked decisions from that feature]
     What struggled: [bullet list from learning report]
     Known risks not fully resolved: [if any]

   [Repeat for each relevant prior feature]

   These learnings will seed the research and PRD. Tell me if any are not applicable to what you are building.
   ```

4. Note which prior decisions are "already locked" for this feature area (e.g., "we always use cursor pagination for this entity"). These will be pre-populated in the decision log sidecar (`decisions/[feature]-decision-log.md`) later — do not re-debate them during discovery unless the PM explicitly raises a change.

5. **Optional: query Jira for related existing work.** If the PM has provided a Jira project at intake (see CLAUDE.md Intake Parameters), ask:

   "Should I check Jira for existing tickets or Epics in [project] that might be related to this feature? This takes 10 seconds and often surfaces scope the PRD should reference or avoid duplicating."

   If yes, run:
   ```
   searchJiraIssuesUsingJql:
   project = [PROJECT] AND (summary ~ "[feature keyword]" OR labels in ([relevant area label]))
   ORDER BY created DESC
   ```

   Surface any matching Epics or Stories:
   ```
   Related existing Jira work found:
   - [EPIC-KEY] — [Epic title] (status: [Open/Done], [date])
     [One-line summary of what it covers]
   - [STORY-KEY] — [Story title] (status, date)
   ```

   Ask the PM: "Are any of these related? If so, should this feature build on, replace, or avoid overlapping with them?"

   If no Jira project is known, or PM says no, skip silently.

6. If the knowledge base does not exist and no Jira results are found, skip this step silently and proceed to Stage 1.

---

# Stage 1 — Interactive Discovery (Mandatory)

Goal: build a complete understanding of the idea before doing any research. The PM may not surface everything you need — your job is to probe for missing detail iteratively until you have enough.

Do not stop at a fixed number of questions. Stop when every critical dimension on the internal checklist (Step 1.2) is covered to a usable level of detail.

---

## Step 1.1 — Open the floor

Ask the PM exactly:

> "Describe the idea in your own words. As much or as little detail as you want — I'll probe for the rest."

Read the full description before doing anything else.

Capture the PM's exact phrasing for the problem, the user, and the desired outcome. This verbatim language seeds the **Voice of Customer** section in Step 1.6. Do not paraphrase yet.

---

## Step 1.2 — Internal dimension checklist (model-private)

You hold this checklist privately. The PM does not see it. It is your forcing function for completeness.

15 dimensions, grouped by purpose:

**Who (4)**
1. **Primary user** — role + sophistication + context. Specific enough that you could find one.
2. **Secondary users / stakeholders affected** by the feature.
3. **Internal political stakeholders** — who else has a stake in the decision (engineering, design, customer success, legal, compliance, leadership)?
4. **Audience size** (10? 1k? 1M?).

**Problem (5)**
5. **The exact verb-level task** being done.
6. **Current workaround / status quo** — what they do today.
7. **Why the workaround is painful** — 5-whys depth, not just one-line "it's slow."
8. **Frequency + severity** — how often (daily / weekly / monthly) and how bad (mild annoyance / blocking / dealbreaker).
9. **Trigger / context** — what makes them need this in the moment?

**Business (3)**
10. **Business motivation** (revenue / retention / activation / differentiation / cost reduction / risk / compliance).
11. **Success signal** — how the user knows it worked.
12. **Failure modes** — could ship perfectly and still fail; why?

**Boundaries (3)**
13. **Adjacent tools** the user uses (what does this fit between in their day)?
14. **Constraints** (timeline / budget / team capacity / regulatory).
15. **Anti-goals** — what we deliberately won't do.

After reading the PM's description in Step 1.1, mentally mark each dimension: **Covered / Partially covered / Missing**.

---

## Step 1.3 — Acknowledge what's clear, surface what's missing

Before asking new questions, tell the PM what you already have:

> "From what you described, I have a clear picture of [list the covered dimensions in plain language]. I'd like to dig deeper on [list the gaps]."

This makes the PM feel heard. It also surfaces your interpretation early so the PM can correct anything you misunderstood.

---

## Step 1.4 — Iterative probing

Ask 3–5 clarifying questions at a time, prioritized by:

- **Critical missing dimensions first** (especially the dimensions PMs systematically forget — see below). Nice-to-have dimensions later.
- **Group related questions** — user questions together, problem questions together, business questions together. Don't jump around.
- **Use 5-whys depth on the problem statement specifically.** If the PM says "users want a dashboard," ask: why? What does the dashboard let them do that they can't today? What changes if they have it? Why does that matter?
- **Ask for a real story or specific example.** "Tell me about the last time a user hit this problem." Anecdotes surface details abstractions miss.

After each round of answers, re-mark the dimension checklist. Loop until no critical dimensions are missing. No fixed question count — could be 1 round, could be 4.

### Specifically probe for dimensions PMs systematically forget

These come up rarely on their own. Probe explicitly:

- **Anti-goals**: "What is this explicitly NOT? What would we deliberately leave out?"
- **Audience size**: "How many users would touch this? 10? 1,000? 100,000?"
- **Secondary stakeholders**: "Beyond the primary user, who else has a stake — engineering, design, customer success, legal, compliance?"
- **Internal political stakeholders**: "Is there anyone whose buy-in is required, or who could block this?"
- **Failure modes**: "If we shipped this perfectly and adoption failed, why would that be?"
- **Falsifiability**: "How would we know if we got the problem wrong?"

---

## Step 1.5 — Handle "I don't know" gracefully

If the PM doesn't know an answer:

- **Don't guess on their behalf.** Never silently fill in a dimension the PM left blank.
- Offer two options:
  1. Mark the dimension as an **open question** that the PRD will surface for stakeholder input later.
  2. Propose **1–2 reasonable assumptions**, explain the tradeoffs of each, and ask which to use as a working assumption.
- If the PM picks option 2, record the choice as an explicit assumption in the discovery output so it can be revisited.

---

## Step 1.6 — Voice of Customer capture

Build a short Voice of Customer section using the PM's verbatim language from Step 1.1 (and any especially vivid phrasing from probing rounds). Hold this in conversation state and reference it downstream.

```
## Voice of Customer (verbatim from PM)

**How the PM described the problem:**
"[exact phrasing from Step 1.1]"

**Words the PM used for the user:**
"[exact phrasing]"

**Words the PM used for the desired outcome:**
"[exact phrasing]"

**A specific example the PM gave (anecdote):**
"[exact phrasing]"
```

This raw language seeds UI copy, user story wording, and acceptance-criteria phrasing in downstream steps. Do not paraphrase. Do not sanitize. Do not "improve" the language.

---

## Step 1.7 — Completeness check before transitioning

Before proceeding to market scan, summarize your understanding back to the PM in a single short block:

```
Here's what I've got:
- **Primary user:** [...]
- **Secondary users / stakeholders:** [...]
- **Problem (verb-level):** [...]
- **Current workaround + why painful:** [...]
- **Frequency + severity + trigger:** [...]
- **Business motivation:** [...]
- **Anti-goals:** [...]
- **Audience size:** [...]
- **Constraints:** [...]
- **Failure modes:** [...]
- **Success signal:** [...]

Confirm this is accurate, or correct me before I proceed to market scan.
```

Wait for explicit confirmation or correction. Do not proceed on silence.

Only after the PM confirms, state:

> "Discovery complete. Proceeding to market scan."

---

## Optional Step 1.8 — Visual sketch (for visual-thinker PMs)

After the PM confirms the summary in Step 1.7, optionally offer:

> "Want to sketch the current user flow or draw the desired flow? Some PMs find drawing surfaces things words miss. Paste an image, describe a flow in steps, or skip."

If the PM provides a sketch or flow description, incorporate it into the discovery output. If they decline, skip.

---

## Stop condition

All of the following must be true before transitioning to Stage 2:

- Every dimension on the internal checklist is **Covered** (not Missing, not Partially covered).
- The PM has confirmed the summary in Step 1.7.
- The Voice of Customer block has been captured.
- Any "I don't know" answers have been explicitly marked as open questions or working assumptions.

---

# Stage 2 — Market & Competitor Scan (Mandatory)

Identify whether competitors or similar solutions exist.

**Use web search for current competitive data.** Do not rely only on training knowledge, which has a cutoff date. Run targeted searches before building the comparison table:

```
Search queries to run (adapt to the actual feature):
1. "[feature name] competitors" OR "[feature name] alternatives"
2. "[primary user type] [job to be done] tools [current year]"
3. "[problem space] product comparison site:g2.com OR site:capterra.com"
```

Run these searches first. Use the results to populate the comparison table. Note the search date in the table caption so the PM knows the data is current.

If search tools are unavailable (no web access), proceed with training knowledge and note: "Competitive data from training knowledge (no live web access). Verify before using in investor or stakeholder materials."

---

If competitors exist, produce a structured comparison table:

| Competitor | Core Offering | Target User | Strengths | Weaknesses | Monetization Model | Opportunity Gap |

The table is mandatory if competitors exist.
Do not skip.

Then summarize:

- Market patterns
- Over-served areas
- Under-served areas
- Strategic whitespace
- Differentiation vectors

If no direct competitors exist:

- Identify adjacent substitutes
- Analyze category creation risk
- Identify adoption friction risks

Explicitly state:
"Market scan complete. Proceeding to strategic evaluation."

Only then continue.

---

# Stage 3 — Strategic Evaluation

Provide structured analysis:

## Why This Could Win
- Structural advantages
- Timing factors
- Distribution leverage
- Network effects (if any)

## Why This Could Fail
- Adoption barriers
- Competitive response risk
- Technical feasibility risks
- Monetization risk

## Required Advantage
What must be true for this to succeed?

## Complexity Assessment
- Technical complexity: Low / Medium / High
- Distribution complexity: Low / Medium / High
- Organizational complexity: Low / Medium / High

---

# Stage 4 — 10x Improvement Test

For existing competitors:

- What would make this 10x better instead of 10% better?
- What would remove friction entirely?
- What would create switching momentum?

If no competitors:
- What would make this defensible from day one?

---

# Stage 5 — Decision Gate (Mandatory)

Ask:

- Refine the idea?
- Kill the idea?
- Proceed to structured PRD using /create-prd?

Do not automatically proceed to PRD.
Wait for explicit user direction.