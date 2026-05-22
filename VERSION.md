v2.9.0 — 2026-05-22

Two new semantic content validators:

- `/validate-prd` — six checks on the PRD: internal consistency (sections contradicting each other), hallucinated data (claims without sources), completeness ([TBD]/[TODO]/empty), VOC traceability (PRD reflects actual research vs. generic AI prose), NFR measurability (concrete thresholds vs. vague aspirations), scope coherence (in-scope vs. out-of-scope contradictions).

- `/validate-user-stories` — seven checks on the breakdown: story↔PRD traceability, AC duplication / contradiction across stories, FE/BE pair coherence (endpoint contracts + naming + permissions), AC specificity (catches vague Gherkin), sizing sanity (scenario count vs. size label), DRAFT consistency (state ↔ breakdown agree), wave / dependency sanity (acyclic + correctly ordered).

Different from `/pipeline-doctor`: the doctor catches structural drift (missing files, schema problems); these validators catch content drift (a PRD that no longer agrees with itself after 17 review-and-fix cycles; a 75-story breakdown where two stories specify different copy for the same screen; an L-sized story that's actually 3 trivial scenarios).

Same UX as the doctor — inline summary, per-finding fix approval, timestamped markdown report to [feature]/validation/.

Cost note: /validate-user-stories is the most expensive command in the skill. On a 614KB breakdown like nestfully-ai, expect 1–3 minutes runtime.

See CHANGELOG.md for full version history.
