v2.10.0 — 2026-05-22

Wave sequencing added to the User Stories Breakdown step.

The /user-stories step (and Step 10 of /build-product) now computes a Wave assignment for every story via topological sort on the Depends On column. Wave 1 = stories with no dependencies; Wave N = stories whose dependencies are all in Waves 1..N-1. Global numbering (W1, W2, ... W[N]). Phase ordering respected. FE/BE pairs are NOT treated as hard dependencies (they commonly land in the same wave).

Cycles in the dependency graph are CRITICAL Gate 3 findings — surfaced with the US-IDs in the cycle, refuse to advance until PM resolves.

The Build Sequence Map gains a Wave column; a new Wave Summary table lists theme + story_ids per wave, with annotations for the critical convergence wave and the launch gate wave. State schema adds user_stories.waves[]. Two new Gate 3 quality checks: every_story_has_wave, dependency_graph_acyclic.

Closes the loop with /validate-user-stories Check 7 (which already expected waves to exist).

See CHANGELOG.md for full version history.
