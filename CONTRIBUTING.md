# Contributing

## Scope

This is a skill — Markdown instructions Claude reads at runtime, not executable code.
Changes are edits to `SKILL.md` and the files under `references/`.

The bar for adding text is high. Every line competes for the model's attention, so a
change should either fix wrong behavior or prevent a real failure. "More detail" on its
own usually makes a skill worse.

## Structure

`SKILL.md` is always loaded. It holds the decision flow, the confirm gate, and rules
that apply to every invocation. Keep it short.

`references/*.md` load on demand. Detail belongs here — a quick "comment on PROJ-12"
shouldn't pull in plan-decomposition guidance.

## Rules

**Never hardcode instance-specific values.** No cloudIds, project keys, or
`customfield_*` IDs anywhere in the repo. These differ per Jira site and are the main
reason automation like this fails silently. Everything instance-specific comes from
runtime discovery (`references/setup.md`) and lives in the user's local config.

Use `PROJ-123` and `PROJ` in examples.

**Keep the confirm gate intact.** Reads are free; anything that writes to a team's
board needs explicit user confirmation. Don't add paths that bypass it.

**Verify API claims against a real instance.** If you're documenting a field, type
name, or error message, confirm it from an actual Jira response rather than memory.
Note in the PR which Jira configuration you tested against — company-managed and
team-managed projects behave differently, and that difference has caused most of the
bugs this skill guards against.

## Testing a change

There's no test suite. Install locally and exercise the paths you touched:

```bash
cp -r skills/jira-automation ~/.claude/skills/
```

Worth checking against both a company-managed and a team-managed project, since their
field configs diverge.

## Reporting a problem

Include the Jira project style (company-managed or team-managed), the exact error text,
and the tool call that produced it. Redact your cloudId and project keys.
