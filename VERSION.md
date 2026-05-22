v2.8.1 — 2026-05-22

PATCH — DRAFT-mode Gate 2 now records a distinct "Deferred" state instead of being indistinguishable from "haven't reached Gate 2 yet".

Pre-v2.8.1, when the PM proceeded past Step 9 in DRAFT mode (no finalized designs), `gates.gate_2` was left as `"Pending"`. That looked identical to "this gate hasn't been reached yet" and caused `/pipeline-doctor` to falsely flag the feature as state-corrupted.

v2.8.1+ writes `gates.gate_2 = "Deferred — DRAFT mode (no finalized designs as of YYYY-MM-DD)"` instead. Gate 2 will still be formally approved later when designs arrive. The doctor's B4 check now recognizes the deferred pattern and downgrades it to INFO instead of WARNING.

See CHANGELOG.md for full version history.
