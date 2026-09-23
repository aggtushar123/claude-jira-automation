# Changelog

All notable changes to this project are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning follows [SemVer](https://semver.org/spec/v2.0.0.html).

## [1.1.0] — 2026-09-23

### Added
- **User onboarding — there is no default user.** A fresh install is bound to no
  account. Creating, commenting, and transitioning are blocked until the person using
  the skill authenticates and explicitly claims the connected identity.
  `references/onboarding.md` carries the flow: detect the connector, connect it if
  absent, show the authenticated account in full, and require confirmation.
- Per-session identity check. `atlassianUserInfo()` is compared against the recorded
  binding before the first write. Silent on match.
- `config.account` — `null` until onboarding completes, then the claimed account's ID,
  display name, and email. Only `onboarding.md` may set it.
- `config.defaultAssignee` — optional, `null` by default, applied as
  `assignee_account_id` on create. A convenience for auto-assignment, explicitly not an
  identity workaround.
- Troubleshooting entries for a blocked unonboarded install, tickets filed under the
  wrong name, and re-authentication that silently lands on the same account.

### Changed
- Config schema is now `version: 2`. Version 1 configs remain readable; a missing
  `account` key is treated as `null`, triggering onboarding without discarding project
  metadata.
- A connector whose account no longer matches the binding voids it — `account` is reset
  to `null` and the user onboards again. There is deliberately no "continue anyway"
  path.
- `setup.md` steps renumbered 1–5, with onboarding as the gate ahead of discovery.
- Sanity check in `troubleshooting.md` now starts with the identity call.

### Notes
- The Jira API provides no way to set an issue's reporter; it is always the
  authenticated account. This release refuses to guess who that should be rather than
  working around the constraint, because most Jira roles cannot correct a reporter
  after creation — the ticket has to be closed and refiled.
- The skill cannot perform the OAuth step itself. It detects connector state and guides
  the user to `/mcp` or claude.ai connector settings.

## [1.0.0] — 2026-01-15

Initial release.

### Added
- Plan decomposition into linked epic / story / subtask hierarchies
- Ticket creation from live git state — working diff, branch, staged changes
- Duplicate guard: scans branch name and recent commits for an existing issue key
  before creating anything
- Bug workflow with symptom-first summaries, stack trace capture, and
  `file:line` root-cause references
- Runtime instance discovery cached to `~/.claude/jira-automation/config.json` —
  no hardcoded custom field IDs
- Comment, transition, and edit support for existing tickets
- Branch name and commit subject suggestions for Jira development-panel linking
- Troubleshooting reference mapping Jira API errors to causes
- Packaged as a Claude Code plugin with marketplace manifest

### Notes
- Jira Cloud only; Server and Data Center are not supported
- Requires the Atlassian MCP connector with `write:jira-work` scope
