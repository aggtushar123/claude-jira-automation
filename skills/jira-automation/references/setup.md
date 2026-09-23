# First-run setup

Jira instances disagree with each other in ways that fail quietly. Story Points is
`customfield_10016` on one site and `customfield_10033` on another. One project calls
it `Sub-task`, the next calls it `Subtask`. A team-managed project may have no
priority field at all. Guess wrong and you get a `201 Created` with a blank field, or
a `400` with an unhelpful message.

So resolve it once, from the API, and cache the answer.

Run this when `~/.claude/jira-automation/config.json` is missing, or when the user
targets a project that isn't in it yet. Every step is read-only until the final write.

## 1. Onboard the user

Do this **first**, before discovering anything else. Everything below is wasted work if
nobody is onboarded, and no write is permitted until somebody is.

Run `references/onboarding.md` in full. It connects the connector if needed, shows the
authenticated identity, and requires the user to claim it explicitly.

Come back here only when onboarding wrote a non-`null` `account`. If it ended with
`account` still `null` — the user said it isn't them, or the connector isn't connected
— **stop**. Don't discover projects for an account nobody has claimed.

## 2. Find the site

```
getAccessibleAtlassianResources()
```

Returns the sites this account can reach. Each has an `id` (the cloudId), `url`, and
`name`. One site → use it. Several → ask which one; don't assume.

An empty result here, after step 1 succeeded, means the account is authenticated but
has no site authorized — the grant was dropped or never completed. Reconnect.

## 3. List projects

```
getVisibleJiraProjects(cloudId=..., action="create", expandIssueTypes=true)
```

`action="create"` matters: it returns only projects the user can actually create
issues in. For each project record `key`, `id`, `name`, and `style`
(`classic` = company-managed, `next-gen` = team-managed).

From `issueTypes`, map each level by `hierarchyLevel`, **not** by name:

| hierarchyLevel | Meaning  |
|----------------|----------|
| `1`            | Epic     |
| `0`            | Story, Task, Bug — the standard level |
| `-1`           | Subtask (`subtask: true`) |

Record the **exact `name` string** for each type you'll use. That string is what gets
passed as `issueTypeName`, and its spelling varies between projects on the same site.

Not every project has every type. A project with no Story type needs Task instead;
a project with no Bug type files fixes as Task. Record what's actually there.

## 4. Read field metadata

For each project the user is likely to use, for its story-level type:

```
getJiraIssueTypeMetaWithFields(
  cloudId=..., projectIdOrKey=<key>,
  issueTypeId=<id>, requiredFieldsOnly=false, maxResults=100
)
```

Pull out:

| What | How to find it |
|------|----------------|
| Story points | `schema.custom` ends in `jsw-story-points`, or name contains "Story point" |
| Sprint | `schema.custom` ends in `gh-sprint` |
| Acceptance criteria | a field whose name matches it — often absent |
| Priority | `fieldId == "priority"`; record `allowedValues[].name` |
| Components | `fieldId == "components"`; record `allowedValues[].name` |
| Required fields | anything with `required: true` and no `hasDefaultValue` |

Match custom fields by `schema.custom` or name, **never** by memorizing an ID.

This is not a hypothetical. `customfield_10033` is Story Points on some instances and
Atlassian's built-in **Design** field on others — same ID, unrelated field, and writing
a number into it either fails or lands somewhere nobody looks. Two projects on the
*same site* can also disagree. Read the ID, don't recall it.

Two things worth noting explicitly because they're the usual silent failures:

- **A missing `priority` field.** If `priority` isn't in the list, this project has
  none. Record `hasPriority: false` and never send it.
- **Empty `allowedValues` on `components`.** The field exists but has no options
  defined. Sending any component name fails.

Repeat for the bug-level type if the project has one — Bug often has `environment`
and `versions` that Story doesn't.

## 5. Write the config

Create `~/.claude/jira-automation/config.json`. The `account` block below is filled in
because step 1 completed — before onboarding, `"account": null` is the correct and only
valid state:

```json
{
  "version": 2,
  "discoveredAt": "2026-01-15",
  "account": {
    "accountId": "712020:00000000-0000-0000-0000-000000000000",
    "displayName": "Jai Kumar Rathore",
    "email": "jai@example.com",
    "confirmedAt": "2026-01-15"
  },
  "defaultAssignee": null,
  "site": {
    "cloudId": "00000000-0000-0000-0000-000000000000",
    "url": "https://example.atlassian.net",
    "name": "example"
  },
  "defaultProject": "PROJ",
  "projects": {
    "PROJ": {
      "id": "10001",
      "name": "Example Project",
      "style": "company-managed",
      "issueTypes": {
        "epic":    { "name": "Epic",     "id": "10000" },
        "story":   { "name": "Story",    "id": "10006" },
        "task":    { "name": "Task",     "id": "10043" },
        "bug":     { "name": "Bug",      "id": "10339" },
        "subtask": { "name": "Sub-task", "id": "10044" }
      },
      "fields": {
        "storyPoints": "customfield_10016",
        "sprint": "customfield_10020",
        "acceptanceCriteria": null
      },
      "hasPriority": true,
      "priorities": ["Highest", "High", "Medium", "Low", "Lowest"],
      "components": [],
      "requiredFields": ["summary"]
    }
  }
}
```

The IDs above are illustrative. `customfield_10016` and `customfield_10020` happen to
be Atlassian's stock defaults for Story Points and Sprint, but plenty of sites differ —
write what step 4 actually returned, never what this example shows.

Conventions:

- `null` means "this instance doesn't have it" — a real answer, not a gap. Don't
  re-discover a `null`.
- Omit an `issueTypes` entry entirely when the project lacks that type.
- `defaultProject` is optional. Set it only if the user names one; otherwise ask each
  time rather than guessing.
- `account` is `null` until onboarding completes. `null` means **no user is bound and
  no write is allowed** — it is never a value to fill in from context, a git author, or
  a name that looks right. Only `references/onboarding.md` may set it.
- `defaultAssignee` is `null` unless the user explicitly asked for tickets to be
  auto-assigned. It's a convenience, unrelated to identity, and never a workaround for
  the wrong account being connected.

The config holds no credentials. Account ID, display name, and email are the same
details visible on any Jira issue that account has touched.

Tell the user what you found and where you saved it. Keep it to a few lines — the
confirmed account, the site, the projects, and anything surprising (a missing priority
field, an unusual type spelling). This is also the moment to ask whether one project
should be the default.

## Re-running

Safe at any time. Triggers:

- A target project isn't in the config
- `atlassianUserInfo()` returns an account that doesn't match `config.account` — set
  `account` to `null` and re-onboard before anything else
- The config has no `account` key at all (written by version 1 of this skill) — treat
  it as `null`: onboard, then bump `version` to 2 and leave project data alone
- A create fails with an unknown-field or invalid-value error
- Someone changed the Jira project's fields or workflow
- `discoveredAt` is old enough to distrust

Re-discover only the affected project and merge — don't discard config for projects
that still work.

## Adding a project later

Same steps 3–4 for the one project, then merge its entry into `projects`. No need to
re-read the whole site.
