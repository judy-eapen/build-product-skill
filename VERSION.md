v2.8.0 — 2026-05-22

Confluence publish scope tightened. Three artifacts are no longer published as Confluence child pages:

- Step 4a: Product Review — internal artifact, kept local.
- Step 4b: Technical Review — internal artifact, kept local.
- Full User Stories Breakdown with Gherkin AC — too large for Confluence; already attached to each Jira Epic.

Step 10 on Confluence is now a lightweight Jira-index page instead of the full breakdown: per-Epic block with title, theme, story count, Jira Epic URL, and story titles only (no Gherkin AC). Stakeholders use it for navigation; the full detail lives in Jira.

Existing pages from pre-v2.8.0 publishes are NOT auto-deleted (bookmarks preserved). Legacy state entries are gracefully ignored.

See CHANGELOG.md for full version history.
