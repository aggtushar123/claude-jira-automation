# Jira Automation for Claude Code

Create Jira tickets from the work you're already doing — a design doc you just wrote,
or the diff sitting in your working tree — without leaving the terminal.

```
> file a bug for what we just fixed

Checked branch and recent commits — no existing issue key.

  PROJ  Example Project
  BUG   Checkout total ignores expired discount codes        High

  Root cause: discount validity check compared against cart
  creation time instead of checkout time (pricing.ts:142)

Create this?

> yes

PROJ-418 created — https://example.atlassian.net/browse/PROJ-418
Commit with:  git commit -m "PROJ-418 Fix discount expiry comparison"
```

## Why

The decomposition is the hard part, and it's already done by the time you open Jira.
What's left is transcription: copying summaries you wrote into description fields,
re-entering acceptance criteria, linking epics to stories across three screens.

Worse, that transcription happens *after* the planning session ends — so you end up
filling in fields from a document you wrote twelve hours ago instead of from working
memory. This skill closes that gap. Planning and ticketing happen in the same session.

## What it does

- **Plan → hierarchy.** Paste a design doc or feature brief; get an epic with stories
  and subtasks, correctly linked, with acceptance criteria pulled from the source.
- **Diff → ticket.** File a bug or story from what you actually changed, with the root
  cause citing real file:line references.
- **Won't duplicate.** Checks your branch name and recent commits for an existing issue
  key first. If you're on `PROJ-123-fix-auth`, it comments on that ticket instead of
  opening a second one.
- **Won't file as the wrong person.** No default user — writes are blocked until you
  authenticate and claim the connected account yourself, re-checked every session.
- **Manages existing tickets.** Comments, transitions, edits, JQL search.
- **Confirms before writing.** Nothing reaches your team's board without you seeing it
  first.

## Requirements

- **Claude Code**
- **The Atlassian MCP connector**, connected to a Jira **Cloud** site with
  `write:jira-work` scope — authorized with **your own** Atlassian account, since
  that's the account every ticket will be reported by

Jira Server and Data Center are **not supported** — they use a different auth model
and a field schema this skill doesn't target.

No API token, `.env` file, or shell dependencies. The connector handles auth.

## Install

### As a plugin (recommended)

```
/plugin marketplace add aggtushar123/claude-jira-automation
/plugin install jira-automation@jira-tools
```

### Manually

```bash
git clone https://github.com/aggtushar123/claude-jira-automation.git
cp -r claude-jira-automation/skills/jira-automation ~/.claude/skills/
```

## First run

### Onboarding: there is no default user

A fresh install is bound to **nobody**. It will not create, comment on, or transition
anything until you authenticate and claim the identity yourself. The skill walks you
through it on first use:

```
The Atlassian connector is authenticated as:

  Jane Doe <jane@example.com>

Every ticket this skill creates will permanently carry this account as its
reporter. Jira cannot change a reporter after the fact.

Is this your account?
```

Answer **no** and it stops — no tickets, no discovery, nothing written — and walks you
through re-authenticating (`/mcp`, or claude.ai → Settings → Connectors). There is no
"continue anyway"; using the skill as somebody else is the outcome onboarding exists to
prevent.

This matters because **the connector authenticates as whoever authorized it, not
whoever is typing.** On a machine a colleague set up, or a shared Claude account, the
naive behavior is to silently file everything under their name — and Jira gives most
roles no way to change a reporter afterward. Those tickets have to be closed and
refiled.

Once claimed, the binding is saved and checked each session. If the connector later
changes hands, the binding is void and you onboard again rather than being asked to
wave it through.

### Then it discovers your instance

Once you're onboarded, the skill caches what it finds to
`~/.claude/jira-automation/config.json`:

- The account you claimed — `null` until you do
- Your cloudId and site URL
- Which projects you can create issues in
- The exact issue type names per project
- Custom field IDs for story points, sprint, acceptance criteria
- Which projects have a priority field, and what values it accepts

This takes a few read-only API calls. You'll be asked to confirm before it writes.

**This step is the whole point.** Custom field IDs are assigned per-instance — Story
Points might be `customfield_10016` on your site and `customfield_10033` on someone
else's. Automation that hardcodes them returns `201 Created` with a blank field, and
you don't notice until standup. The config is discovered, never assumed.

The config is machine-local and gitignored. It holds no credentials — just IDs, field
names, and the account you confirmed.

## Usage

Just describe what you want. The skill triggers on intent, not on a command.

| You say | What happens |
|---|---|
| "create tickets from this plan" *(paste doc)* | Epic + stories + subtasks, linked |
| "file a bug for what we just fixed" | Reads your diff, proposes a Bug |
| "make a ticket before I start the export feature" | Story from your request, offers a branch name |
| "log this stack trace" | Bug with the trace quoted, file:line in root cause |
| "comment on PROJ-45 with what I found" | Proposes comment text, then posts |
| "move PROJ-88 to Done" | Looks up valid transitions, handles subtasks first |

### Linking commits

After creating a ticket the skill offers a commit subject line:

```
PROJ-418 Fix discount expiry comparison
```

Leading with the key is what makes Jira's development panel pick up the commit. It
won't rewrite commits you've already made — it comments the SHA on the ticket instead.

## How it works

```
  Your words                Claude                      Jira Cloud
  ─────────                 ──────                      ──────────
  design doc     ──►   decompose & classify   ──►
  git diff       ──►   check for existing key ──►   Atlassian MCP
  "file a bug"   ──►   draft ticket bodies    ──►    (connector)
                              │
                              ▼
                       show proposal
                              │
                        ◄── you confirm
                              │
                              ▼
                       create top-down:
                       epic → story → subtask
```

Claude does the judgment — what's an epic vs. a story, how to split ambiguous work,
what the acceptance criteria actually are. The connector does execution. The config
file keeps instance-specific facts out of both.

Issues are created parent-first so each child has a real key to link against.

### Files

```
skills/jira-automation/
├── SKILL.md                      entry point, onboarding + confirm gates, creation rules
└── references/
    ├── onboarding.md             authenticate the user; blocks writes until bound
    ├── setup.md                  discovery routine + config schema
    ├── plan-parsing.md           epic/story/subtask heuristics, templates
    ├── dev-workflow.md           tickets from diffs, branches, stack traces
    └── troubleshooting.md        Jira API errors → causes → fixes
```

Reference files load on demand, so a quick "comment on PROJ-12" doesn't pull in the
plan-decomposition guidance.

## Safety

Read-only operations — checking your branch, reading a diff, fetching a ticket,
searching — run without prompting.

Anything your team can see requires explicit confirmation: creating issues, posting
comments, transitioning tickets. Jira has no hard delete, so a wrong ticket is a
cleanup task for a human.

Identity is bound by onboarding and re-checked before the first write of each session,
because a ticket filed under the wrong reporter can't be corrected by most Jira roles —
it has to be closed and refiled. An unonboarded install is inert.

If a batch create fails halfway, the skill reports exactly which keys exist and which
didn't, and resumes from the failure rather than re-running the whole batch.

## Limitations

- **Jira Cloud only.** No Server/DC support.
- **Can't set the reporter.** The Jira API assigns it from the authenticated account.
  Onboarding makes sure you know who that is before anything is written, but if the
  wrong account is connected the only fix is reconnecting as yourself.
- **Can't authenticate for you.** Connecting and re-authenticating the Atlassian
  connector is something you do in `/mcp` or on claude.ai. The skill detects the state
  and guides you; it can't perform the OAuth step.
- **Won't invent required fields.** If your project requires a custom field only a
  human can decide, it asks.
- **Nonstandard workflows and validators will reject tickets.** The skill makes correct
  API calls; it can't work around a project's own rules.
- **Sprint assignment needs a numeric sprint ID**, which isn't discovered
  automatically.
- **Estimates are only as good as the input.** With no sizing signal in the source,
  it leaves story points blank rather than guessing.

## Troubleshooting

Common failures and their causes are in
[`skills/jira-automation/references/troubleshooting.md`](skills/jira-automation/references/troubleshooting.md).

The usual ones:

| Symptom | Cause |
|---|---|
| Refuses to create anything, says you aren't onboarded | Working as designed — no user bound yet |
| Tickets filed under someone else's name | Connector authorized by another account — re-auth via `/mcp` |
| Re-auth keeps landing on the same account | Browser already signed in — use a private window |
| `Field 'customfield_X' cannot be set` | Stale config — re-run discovery |
| `Field 'priority' cannot be set` | Team-managed project with no priority field |
| `Specify a valid issue type` | `Sub-task` vs `Subtask` spelling |
| `Transition id is not valid` | Transition IDs are per-workflow; never hardcoded |
| No Atlassian tools available | Connector isn't connected |

## Credits

The idea — and the architectural split between model-side decomposition and
deterministic execution — comes from Vicky Pandey's
[ai-jira-automation](https://github.com/dev-vpandey/ai-jira-automation) and the
accompanying [write-up](https://levelup.gitconnected.com/i-automated-the-overhead-that-wasnt-actually-engineering-eeb20812e028).

That project targets Jira Server/Data Center through a bash and curl layer. This one
keeps the architecture but retargets it at Jira Cloud via the Atlassian MCP connector,
and replaces hardcoded field mappings with runtime discovery.

## License

MIT — see [LICENSE](LICENSE).
