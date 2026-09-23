# Troubleshooting

Jira's create API fails in a small number of recognizable ways. Match the error before
retrying — a blind retry sends the same bad payload.

## Field errors

**`Field 'customfield_XXXXX' cannot be set. It is not on the appropriate screen, or unknown.`**

The field ID is wrong for this project, or the field isn't on the create screen. Most
common cause: an ID copied from another instance or another project. IDs collide
across sites — `customfield_10033` is Story Points on some and the built-in Design
field on others — so a copied ID can succeed and write to the wrong field. Re-run discovery
(`setup.md` step 4) for this project and use what it returns. If the field genuinely
isn't on the create screen, create the issue without it and set it with `editJiraIssue`
afterward.

**`Field 'priority' cannot be set.`**

This project has no priority field — common in team-managed projects. Set
`hasPriority: false` in the config and drop the field.

**`Specify a valid 'id' or 'name' for priority`**

The priority scheme differs. Instances using `P0`–`P3` and instances using
`Highest`–`Lowest` are both common. Use a value from the project's discovered
`priorities` list.

**`Component name 'X' is not valid`**

Either the component doesn't exist, or the project has components enabled with none
defined (`allowedValues: []`). Omit the field.

**`Field 'summary' is required`** — plus other required fields you didn't send.

Some projects add required custom fields. Discovery records these in `requiredFields`;
if it's a field only a human can decide, ask rather than inventing a value.

## Issue type errors

**`Specify a valid issue type`** / **`issuetype is required`**

The type name string is wrong for this project. `Sub-task` and `Subtask` are both real
spellings on different projects — sometimes on the same site. Use the exact `name`
from discovery. Also check the type exists at all: not every project has Story or Bug.

## Parent errors

**`Field 'parent' cannot be set`** on a story

The project may not support epic-parent linking, or the type hierarchy is unusual.
Create the story unparented and link it afterward.

**`Parent issue is not a valid parent`**

Wrong level. A subtask's parent must be a story-level issue, never an epic. A story's
parent must be an epic.

**`Issue does not exist or you do not have permission to see it`**

The parent key is wrong, or the parent create silently failed earlier in the run. Stop
and check what was actually created before continuing.

## Auth and identity

**Tickets are created under the wrong person's name**

The connector is authenticated as whoever authorized it, and the create API offers no
way to set a reporter. A machine someone else configured, a shared Claude account, or a
browser still signed in as a colleague all produce this. Most Jira roles cannot change
a reporter after the fact, so the tickets usually have to be closed and refiled.

Onboarding exists to make this impossible — writes are blocked until the user claims
the authenticated identity. Reaching this state means onboarding was bypassed, or
`config.account` was filled in from something other than
`references/onboarding.md`.

Fix: set `config.account` to `null`, then run `references/onboarding.md` from the top.
Don't patch the account block by hand — the point of the flow is the user's explicit
confirmation, and editing the file around it recreates the bug.

**Skill refuses to create anything, says the user isn't onboarded**

Working as designed. `config.account` is `null`, so no user is bound. Run
`references/onboarding.md`. This is also the expected state right after a connector is
disconnected and reconnected.

**Onboarding keeps showing the same wrong account after re-authenticating**

The browser was already signed in to Atlassian as that person, so the OAuth
re-authorization completed silently without a login screen. Disconnect again and
reconnect from a private window.

Also confirm Claude Code was restarted — the connector identity is cached for the
session.

**No Atlassian tools available**

The connector isn't set up. See the README — this skill has no fallback path.

**`atlassianUserInfo` errors or returns nothing**

Not connected, or the token expired. Reconnect before anything else — every other
call will fail too.

**`getAccessibleAtlassianResources` returns `[]`**

Connected but no site authorized, or the token lost its grant. Reconnect.

**`401` / `403` on create, reads work fine**

The account can view but not create in this project. Confirm with the Jira admin — this
isn't something the skill can work around.

## Transitions

**`Transition id X is not valid`**

Transition IDs are per-workflow and not stable across projects. Always call
`getTransitionsForJiraIssue` for the specific issue first. There is no universal
"Done" ID.

**Parent won't close**

Many workflows block closing a parent with open subtasks. Transition the children
first.

## Partial runs

A multi-issue create that fails halfway leaves real tickets behind. Report exactly
which keys exist and which didn't get created. Don't re-run the whole batch — that
duplicates the successful half. Resume from the failure.

## Sanity check

To confirm the connector and config still line up:

1. `atlassianUserInfo()` — does `account_id` still match `config.account.accountId`?
2. `getAccessibleAtlassianResources()` — does the cloudId still match the config?
3. `getVisibleJiraProjects(action="create")` — is the target project still listed?
4. `getJiraIssueTypeMetaWithFields(...)` — do the cached field IDs still appear?

A mismatch at any step means re-run discovery for that project.
