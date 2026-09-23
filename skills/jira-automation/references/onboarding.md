# Onboarding: authenticating the user

**This skill has no default user.** A fresh install is bound to nobody, and it stays
that way until the person using it authenticates and claims the identity themselves.
Until that happens, every write — create, comment, transition — is blocked.

The reason is narrow and non-negotiable: Jira takes the **reporter** from the
authenticated account, offers no API to set it, and gives most roles no way to change
it after creation. A ticket filed under the wrong name isn't a bad field value, it's a
ticket that has to be closed and refiled. Inheriting whatever account happens to be
connected is how that happens, so the skill doesn't inherit.

## When to run this

- `~/.claude/jira-automation/config.json` does not exist
- It exists and `account` is `null`
- It exists and `account.accountId` no longer matches `atlassianUserInfo()`

The third case is not a warning. The binding is broken — the connector now belongs to
someone else. Set `account` back to `null` and onboard from the top.

## Step 1: is the connector there at all?

```
atlassianUserInfo()
```

**Errors, or returns nothing** → the Atlassian connector isn't connected. The skill
cannot connect it; the user has to. Tell them:

1. `/mcp` in Claude Code lists MCP servers and connectors. If Atlassian appears there,
   authenticate it from that menu.
2. If it doesn't, the connector is managed on claude.ai — Settings → Connectors → add
   Atlassian, and authorize a Jira **Cloud** site with `write:jira-work` scope.
3. Restart Claude Code, then start over at step 1.

Stop here until it returns an account. There is no fallback path and nothing useful to
do in the meantime.

## Step 2: show the identity, and require a claim

The connector answered, so *some* account is authenticated. That is not yet the user.

Show exactly who it is — full name and email, not a summary — and state the
consequence before asking:

```
The Atlassian connector is authenticated as:

  Jane Doe <jane@example.com>

Every ticket this skill creates will permanently carry this account as its
reporter. Jira cannot change a reporter after the fact.

Is this your account?
```

Rules for this exchange:

- **Ask, and wait.** Do not answer it from the git author, the Claude account email,
  the repo owner, or anything else you happen to know. Those are different identity
  systems and they disagree routinely — matching names is not authentication.
- **A bare "yes" is the only pass.** Silence, "go ahead", "just make the ticket", or
  moving on to a different topic are not confirmation. Re-ask.
- **Don't editorialize toward yes.** Don't say "this looks like you" or note that the
  names match. The user confirms; you don't nudge.

## Step 3a: they confirm it's them

Record the binding and continue to `setup.md` step 2 for instance discovery.

```json
"account": {
  "accountId": "<account_id from atlassianUserInfo>",
  "displayName": "<name>",
  "email": "<email>",
  "confirmedAt": "<today, ISO date>"
}
```

This is written once. Later sessions compare against it and say nothing when it
matches.

## Step 3b: it isn't them

**Stop. Do not offer to proceed anyway, and do not offer to work around it.** There is
no "continue as this user" path, because using the skill as somebody else is the exact
outcome onboarding exists to prevent. The only resolution is re-authenticating.

Walk them through it:

1. `/mcp` in Claude Code — for OAuth connectors this offers to clear authentication
   and re-authenticate. Do that and sign in as themselves.
2. If the connector is managed on claude.ai instead: Settings → Connectors → Atlassian
   → disconnect, then reconnect.
3. **Use a private browser window.** A browser already signed in to Atlassian as the
   other person will re-authorize as them without ever showing a login screen. This is
   the single most common way people "switch accounts" and end up exactly where they
   started.
4. Restart Claude Code so the connector re-syncs.
5. Run onboarding again from step 1.

If the connected account belongs to a colleague on a shared machine, or the Claude
account itself is shared, say so plainly — disconnecting may affect the other person,
and that's worth a conversation with them before doing it.

Leave `account` as `null`. An unonboarded config is the correct state to leave behind;
a half-bound one is not.

## What onboarding does not do

- **It does not grant permissions.** If the account can read but not create in a
  project, that's a Jira admin question. Onboarding confirms identity, not authority.
- **It does not set the reporter.** Nothing can. It confirms who the reporter will be.
- **It does not need re-running every session.** Once `account` is recorded, the
  per-session check is one `atlassianUserInfo()` call and a comparison. Silent on
  match.

## Assignee is a separate question

`defaultAssignee` is an unrelated convenience — "assign new tickets to me by default"
— and it starts `null`, meaning tickets are created unassigned. It is **not** a
workaround for the wrong account being connected, and it must never be offered as one:
assigning a ticket to the right person does not make the wrong person's name on it
correct.

Set it only when the user asks for automatic assignment, using
`lookupJiraAccountId(cloudId, searchString=<name>)`.
