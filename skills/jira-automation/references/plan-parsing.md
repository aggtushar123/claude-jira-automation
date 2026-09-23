# Plan parsing

How to turn prose into a hierarchy. The goal is tickets someone can pick up mid-sprint
without asking a follow-up question.

## What goes at each level

**Epic** — a deliverable a stakeholder would recognize by name, spanning more than one
sprint. Usually the plan's title or a top-level phase heading. Most plans yield exactly
one epic. If you're about to create three epics from one document, they're probably
stories.

**Story** — independently shippable, independently testable, one owner, fits in a
sprint. In a design doc these are the components, services, or phases. A story that
can't be demoed on its own is a subtask.

**Subtask** — a step within a story, hours not days, no separate acceptance criteria.
These are the numbered implementation steps under a component heading.

Signals in the source text:

| In the plan                                  | Level   |
|----------------------------------------------|---------|
| Document title, "Phase N", "Milestone"        | Epic    |
| `##` component/service headings, "Build X"    | Story   |
| Numbered steps, bullets under a component     | Subtask |
| "Also need to…", "Don't forget…"              | Subtask |

## Ambiguity

When something could read either way, prefer the **smaller** unit — a subtask promoted
to a story later is cheap; an epic that should have been a story leaves a board full of
one-story epics.

Don't silently drop work you couldn't classify. Surface it in the proposal as
"unclassified" and ask.

## Estimation

Only estimate when the plan gives you something to estimate from. Fibonacci only:
1, 2, 3, 5, 8, 13. A story landing at 13 is a sign it should be split — say so rather
than filing it.

Leave points off entirely when the plan has no sizing signal. A guessed 5 is worse
than a blank field.

## Description templates

Write Markdown (`contentFormat="markdown"`, the default).

### Epic

```markdown
## Overview
<what this delivers and why, 2–3 sentences>

## Goals
- <outcome, not activity>

## Out of scope
- <explicit non-goals — the part people forget>

## Success criteria
<how we know it's done>
```

### Story

```markdown
**As a** <role> **I want** <capability> **so that** <value>

## Acceptance criteria
- [ ] <observable, testable>
- [ ] <observable, testable>

## Technical notes
<constraints, affected services, data model implications>

## Dependencies
<blocking work, or "None">
```

### Subtask

```markdown
<one-line statement of the change>

## Steps
1. <step>
2. <step>

## Done when
<the check that closes it>
```

Pull real content from the plan into these. An acceptance criterion that just restates
the summary is worse than no acceptance criteria — it's the thing that made manual
ticket entry feel pointless.

## Proposal format

Show this before creating anything:

```
PROJ — Example Project

EPIC  AI Recommendation Engine
  STORY  Build training data pipeline            [8] High
    SUB    Configure Kafka consumer group
    SUB    Add schema registry integration
  STORY  Serve recommendations via API           [5] Medium
    SUB    Define response contract

6 issues: 1 epic, 2 stories, 3 subtasks. Create these?
```

Include the project key and name — it's the field people most often realize is wrong.

Use the type names from the config, not the labels above. If the target project has no
Story type, decompose into whatever its story-level type is (usually Task); if it has
no Bug type, fixes go in as Task too.
