# Changelog

## v8 — Cross-repo + pilot hardening (current)

This version extends v7 with cross-repo aggregation (Option 2 architecture) and
incorporates all fixes discovered during the pilot on PMOTestingOrg.

### New capabilities (cross-repo)
- `project-config.yml` — declares project slug, linked engineering code repos, surfacing toggles, invoice triggers
- `refresh_cross_repo.py` — polls linked code repos and builds dashboard sections (Engineering Activity, Needs PM, High-Severity Bugs, Sprint Status, Recent Closures, Invoice Triggers)
- `sync_labels.py` — one-click creation of `project:<slug>` + supporting labels across linked repos
- `validate_config.py` — validates config, catches renamed/deleted repos, opens/closes a tracking issue
- `audit_unlabeled.py` — daily report of unlabeled engineering issues
- `seed_test_scenario.py` — populates a linked repo with realistic test data for dashboard verification
- PMO GitHub App (`pmo-github-app`) — nudges engineers to apply project labels
- New workflows: `sync-labels.yml`, `validate-config.yml`, `daily-audit.yml`, `seed-test-scenario.yml`
- Dashboard refresh cadence tightened to every 5 minutes

### Bug fixes from pilot

**seed_project.py — owner resolution (org vs user)**
- Symptom: Board cloning failed with "Could not resolve to a User with the login of 'PMOTestingOrg'"
- Cause: `get_owner_node_id()` tried org query first, but the `graphql()` helper raised on any `errors` in the response, falling through to a user query that 404'd for an org
- Fix: Combined `repositoryOwner` query that resolves both org and user in one call, with explicit fallbacks; same approach applied to `get_golden_project_node_id()`

**pmo-github-app/index.js — polling time window**
- Symptom: App never nudged issues older than 30 minutes; manual runs reported "checked 0 new issue(s)"
- Cause: Polling filtered issues to those created in the last 30 min, AND re-checked `createdAt < since` redundantly
- Fix: Removed the time filter entirely; examines all open issues (paginated up to 500), relies on the `<!-- pmo-app:nudge -->` comment marker for dedup so issues are never double-nudged

**update_dashboard.py — GraphQL state casing**
- Symptom: Phase Progress always showed 0% and 0/N closed sub-issues even when sub-issues were closed
- Cause: GraphQL `Issue.state` returns uppercase enum (`OPEN`/`CLOSED`); the codebase compares against lowercase (`open`/`closed`), so nothing matched
- Fix: Normalize parent and sub-issue states to lowercase immediately after the GraphQL fetch

**update_dashboard.py — single vs multiple in-progress phases**
- Symptom: "Currently In" showed only the first open phase even when multiple phases were In Progress on the board
- Cause: `find_current_phase()` returned the first open phase only and ignored the board Status field
- Fix: Added `find_current_phases()` (plural) that reads each phase parent's board Status and lists all phases marked "In Progress"; falls back to earliest open phase if no statuses set

**update_dashboard.py — phase progress ignored board fields**
- Symptom: Progress % didn't reflect the board's `% Complete` field or `Status`
- Cause: Progress was computed only from closed-sub-issue ratio
- Fix: Phase Progress table now has separate Status (from board) and Progress (board `% Complete`, fallback to sub-issue ratio) columns; Status reflects In Progress / Blocked / Not Started / Done

**refresh_cross_repo.py — GraphQL label filter is OR not AND**
- Symptom: "Needs PM Attention" and "High-Severity Bugs" showed ALL project issues, not just those with the additional label
- Cause: GitHub's GraphQL `issues(labels: [...])` uses OR semantics across labels, not AND
- Fix: Query by the project label only, then filter client-side requiring ALL additional labels (e.g., `project:slug` AND `severity:S1`)

**refresh_cross_repo.py — needs-pm surfacing made lenient**
- Change: "Needs PM Attention" now surfaces all `needs-pm` issues in linked repos, split into two subsections — those correctly tagged with the project label, and those missing a project label (flagged so PMs can fix label hygiene). Issues tagged for a different project are excluded.

**refresh_cross_repo.py — rollup column clarity**
- Change: Engineering Activity headers now explicitly state counts are scoped to the project label, to avoid confusion when unlabeled closed issues don't appear in "Closed last 7d"

### Notes / known limitations
- Scheduled GitHub Actions cron is best-effort; auto-polling may lag 10-30 min during peak hours. Manual trigger or webhook mode (App supports `MODE=webhook`) addresses this.
- Engineering Sprint Status shows all open milestones in linked repos, not filtered by project label (engineering's sprint structure is their own organizing concept).
- Items Past Planned End and Phase Progress are PMO-board-side only; they don't reflect engineering code-repo dates.
- The test seeder (`seed_test_scenario.py`) covers cross-repo sections only; PMO-board fields (Currently In, Phase Progress, Items Past Planned End) still need manual verification since Projects v2 field writes are awkward to script reliably.

## v7 — Project-board-first (previous)
- Golden project cloning via `copyProjectV2`
- Native sub-issues via GraphQL `addSubIssue`
- Milestone-based invoicing
- Daily snapshots + weekly timeline
- Dynamic pinning of active phase
