# Creating tickets from work in the terminal

The plan-parsing path assumes a document. Most of the time there isn't one — there's a
branch, a diff, a failing test, or a thing the user just asked for. This is how to get
a ticket out of that.

## Step 0: is there already a ticket?

**Always check before creating.** Duplicate tickets are the main way this skill can
make someone's board worse.

```bash
git branch --show-current
git log --oneline -20
```

Scan both for an issue key — `ABC-123`, matching `[A-Z][A-Z0-9]+-\d+`. Also check
whether the user mentioned one earlier in the session. Cross-check the prefix against
the project keys in the config so a match like `UTF-8` or `SHA-256` isn't mistaken for
a ticket.

If a key turns up, the work is already tracked. Do **not** create anything. Instead:

- `getJiraIssue` to read it and confirm it's really this work
- Comment the progress, or transition it
- Only create something new if the work genuinely escaped that ticket's scope — and
  say so before you do

## Reading the work

Depending on which way the user is facing:

**Work already done** — read the actual change, don't summarize from memory:

```bash
git status --short
git diff --stat
git diff                      # unstaged
git diff --cached             # staged
git log --oneline origin/HEAD..HEAD   # commits not yet pushed
```

The diff is the source of truth for what a retroactive ticket says. If the session
history and the diff disagree, the diff wins.

**Work about to happen** — the source is the user's own request plus whatever you
found while scoping it: the files you read, the constraints you hit, the approach you
settled on. Write the ticket from that, not from a generic restatement of the ask.

## Choosing the type

| The work is…                                      | Type      |
|---------------------------------------------------|-----------|
| Fixing broken behavior                            | `Bug`     |
| New capability, user-visible                      | `Story`   |
| Refactor, upgrade, chore, infra                   | `Task`    |
| A step inside a ticket that already exists        | `Sub-task`/`Subtask` under it |
| A body of work spanning several of the above      | `Epic` + children |

Use the exact type names from the config. Not every project has every type — when
Story or Bug is missing, use the project's story-level type (usually `Task`). Subtask
spelling varies (`Sub-task` vs `Subtask`) even within one site.

One change usually means one ticket. Resist decomposing a 40-line fix into an epic.

## Scale the ceremony to the work

A one-line null check does not need acceptance criteria. Match the ticket to the size
of the change:

- **Small fix** — summary + a short description saying what broke and what fixed it.
  No points, no subtasks.
- **Normal feature or fix** — the full Story/Bug template from `plan-parsing.md`.
- **Multi-part work** — epic with children, the standard propose-the-tree flow.

Over-structured tickets on trivial work are the same overhead this skill exists to
remove.

## Bug descriptions

Write the summary as the **symptom**, not the fix — that's what someone searching the
board will look for. "Checkout total ignores discount codes", not "Add discount lookup
to totals".

```markdown
## What happens
<observed behavior>

## Expected
<what should happen>

## Repro
1. <step>
2. <step>

## Root cause
<what was actually wrong — include `file.ts:42` references>

## Fix
<what changed>
```

Many instances give Bug an `environment` field (plain string, via
`additional_fields`) that Story doesn't have — "staging", "local, Node 20", a browser
version. Use it when discovery found it.

For a bug filed *after* the fix, say so in the description and drop the ticket
straight to the review/done column rather than leaving it open in To Do.

## Reading a stack trace or test failure

When the trigger is failing output, quote the real thing in the description — inside a
fenced block, trimmed to the relevant frames. A ticket with the actual error text is
searchable; a paraphrase isn't.

Pull the file and line out of the trace and name them in **Root cause**.

## Linking the code back to the ticket

After creating, offer these — don't run them unprompted, and never rewrite existing
commits:

**Branch** (for work not yet started):
```bash
git checkout -b PROJ-123-short-slug
```

**Commit subject** — leading the subject with the key is what makes Jira's development
panel pick up the commit:
```
PROJ-123 Fix discount lookup in checkout totals
```

If the user has already committed without the key, leave history alone. Comment the
commit SHA on the ticket instead.

## Confirmation

Everything here still goes through both gates in `SKILL.md` — the identity check
before the first write of the session, then the per-ticket confirm. Show the proposed
ticket — type, project, summary, description — and wait. The user is mid-task; a
surprise ticket on the team board is worse than a slow one.

The exception is the read-only half: checking for an existing key, reading the diff,
and fetching a ticket need no confirmation. Do those freely.

## Worked examples

**"file a bug for what we just fixed"**
→ `git diff` + session context → check branch for an existing key → none → propose
`Bug` in the user's project, symptom summary, root cause citing `auth.ts:88`, fix
described → confirm → create → offer the commit subject line.

**"create a ticket before I start on the export feature"**
→ no diff yet → source is the request + scoping → propose `Story`, acceptance criteria
from what the user described → confirm → create → offer
`git checkout -b PROJ-140-csv-export`.

**"I'm on PROJ-77, log what I found"**
→ key present → `getJiraIssue PROJ-77` → no create → propose comment text → confirm →
`addCommentToJiraIssue`.
