# Changelog

All notable changes to this project are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning follows [SemVer](https://semver.org/spec/v2.0.0.html).

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
