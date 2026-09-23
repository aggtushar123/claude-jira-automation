---
name: jira-automation
description: |
  Create and manage Jira Cloud tickets from inside Claude Code. Works from a pasted plan or design doc, and from live terminal work — the current diff, branch, failing test, or the feature about to be built.

  Triggers: "create JIRA epic/story/subtask", "add to JIRA", "create tickets from plan", "break this into stories", "file a ticket", "log a bug", "make a ticket for this", "ticket for what we just fixed", "track this work", "create a ticket before I start", "add comment to JIRA", "comment on ticket", "change status", "move to Done", "what's in my sprint"
  Examples: "Create tickets from this design doc", "File a bug for the crash we just fixed", "Make a ticket before I start the export feature", "Log this stack trace", "Comment on PROJ-45 with what I found", "Move PROJ-88 to Done"
---

# Jira Automation

Turn work into Jira issues and manage existing ones. Execution goes through the
Atlassian MCP connector (`*Atlassian*` tools — tool names vary by client, commonly
`mcp__atlassian__*` or `mcp__claude_ai_Atlassian_Rovo__*`). There is no API token,
script, or curl layer to configure.

Two entry points, same confirm gate:

- **A plan** — a pasted design doc or brief becomes an epic with stories and subtasks.
- **Live work** — a diff, a branch, a failing test, or a feature about to be built
  becomes a ticket. See `references/dev-workflow.md`; this is the common case when
  working in a repo.

## Before anything else: load config

Jira instances differ in ways that silently break automation — custom field IDs,
issue type naming, which fields a project even accepts. This skill resolves all of
that **once** and caches it.

Read `~/.claude/jira-automation/config.json`.

- **Exists** → use it. Every value you need (cloudId, project keys, field IDs, type
  names, priorities) is in there.
- **Missing, or the user's target project isn't in it** → run
  `references/setup.md` first. It's a few read-only API calls and one file write.

Never hardcode a cloudId, project key, or `customfield_*` ID into a call. If it isn't
in the config, discover it and add it. If a call fails because a field doesn't exist,
the config is stale — re-run discovery for that project rather than guessing.

## From a plan

Creating tickets is a **propose → confirm → execute** cycle. Never create issues
straight from a plan without showing the user the tree first.

1. **Parse.** Read the plan and build a hierarchy. See `references/plan-parsing.md`
   for what counts as an epic vs. story vs. subtask.
2. **Resolve the project.** Use the config's `defaultProject` if set and the user
   didn't name one; otherwise ask. Don't guess between projects — their field configs
   differ.
3. **Find or create the epic.** If the work belongs under an existing epic, search
   before making a duplicate:
   ```
   project = <KEY> AND issuetype = Epic AND statusCategory != Done
   ORDER BY updated DESC
   ```
   Show the candidates and let the user pick, or propose a new epic.
4. **Show the tree and stop.** Render the proposal as an indented list with summaries,
   points, and priorities. Wait for explicit approval. Ticket creation is
   outward-facing and effectively irreversible — Jira has no hard delete here.
5. **Execute top-down.** Epic first, capture its key, then stories with
   `parent=<epic key>`, then subtasks with `parent=<story key>`. A child cannot be
   created before its parent's key exists.
6. **Report.** List every created key as a browse URL:
   `<site url>/browse/<KEY>`

If a create fails mid-run, stop and report what was created and what wasn't. Don't
retry blindly — a 400 almost always means a field this project doesn't accept.

## Working in a repo

When the trigger comes from the terminal rather than a document, the first move is
**not** to create anything.

1. **Check for an existing key.** `git branch --show-current` and
   `git log --oneline -20`. If an issue key appears in either, the work is already
   tracked — comment or transition that ticket instead of opening a duplicate.
2. **Read the actual work.** `git diff`, `git diff --cached`, `git status --short`.
   For work not yet started, the source is the user's request plus what you learned
   scoping it.
3. **Size the ticket to the change.** A one-line fix gets a summary and two sentences,
   not acceptance criteria and subtasks.
4. **Propose, then create.** Same gate as above.

Full guidance — bug descriptions, stack traces, branch and commit linking:
`references/dev-workflow.md`.

Steps 1 and 2 are read-only; do them freely. Creating, commenting, and transitioning
always need confirmation.

## Parent linking

On Jira Cloud there is no Epic Link custom field — it was replaced by `parent`, which
does both jobs:

| Relationship    | How                 |
|-----------------|---------------------|
| Story → Epic    | `parent` = epic key |
| Subtask → Story | `parent` = story key|

A subtask's parent is the **story**, never the epic.

Older guides reference `customfield_10014` ("Epic Link") or an "Epic Name" field. Both
are gone on current Cloud instances. Don't set them.

## Creating issues

Only `cloudId`, `projectKey`, `issueTypeName`, and `summary` are required. Anything
without its own parameter — custom fields, priority, labels, components — goes in
`additional_fields`.

```
createJiraIssue(
  cloudId=<config.site.cloudId>,
  projectKey="PROJ",
  issueTypeName=<config.projects.PROJ.issueTypes.story>,
  summary="Build training data pipeline",
  description="<markdown>",
  parent="PROJ-12",
  additional_fields={
    "<config.projects.PROJ.fields.storyPoints>": 8,
    "priority": {"name": "High"},
    "labels": ["from-plan"]
  }
)
```

Descriptions default to Markdown (`contentFormat="markdown"`). Write real structure —
a user-story line, acceptance criteria, technical notes. Most instances have no
Acceptance Criteria field, so it belongs in the description body unless the config
says otherwise.

Set `priority` only if `config.projects.<KEY>.hasPriority` is true, and only with a
value from that project's `priorities` list. Team-managed projects frequently have no
priority field, and sending one fails the create.

## Existing tickets

- **Comment:** `addCommentToJiraIssue` — show the user the text before posting.
- **Transition:** `getTransitionsForJiraIssue` for valid IDs, then
  `transitionJiraIssue`. Transition names vary by workflow; never hardcode them.
  When closing a parent, check its `subtasks` and handle children first.
- **Edit:** `editJiraIssue`.
- **Read/search:** `getJiraIssue`, `searchJiraIssuesUsingJql`.

Posting a comment or moving a ticket is visible to the whole team — confirm first.

## When something fails

`references/troubleshooting.md` maps the common Jira API errors to causes. Check it
before retrying a failed create.

## References

- `references/setup.md` — first-run discovery, config schema
- `references/dev-workflow.md` — tickets from a diff, branch, bug, or stack trace
- `references/plan-parsing.md` — decomposition heuristics, description templates
- `references/troubleshooting.md` — error → cause → fix
