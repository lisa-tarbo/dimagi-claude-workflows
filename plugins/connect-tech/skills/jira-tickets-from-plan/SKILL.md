---
name: jira-tickets-from-plan
description: Use when creating Jira tickets from a AI plan.
---

# Jira Tickets from AI Plan

## Overview

Reviews an AI plan, breaks it into logically independent tickets, and creates them in Jira. Requires a GitHub link to the plan file — ticket descriptions link back to specific sections of the plan rather than copying the implementation inline.

## Process

### Step 1: Confirm the Plan Source

The user MUST supply a GitHub link to the plan file (ideally a permalink pinned to a commit SHA so line-range anchors stay stable). If they did not, ask explicitly before doing anything else:

> "Please share a GitHub link to the plan file. The ticket descriptions will link to specific sections of the plan, so I need a permalink (pinned to a commit SHA) to ensure the links don't rot."

Do not proceed until a GitHub link is provided.

### Step 2: Review the Plan

Read the full plan. Identify logical breakpoints using these criteria (in priority order):

1. **Independently deployable** — each ticket can be merged and deployed without depending on other in-progress work
2. **Max ~500 lines per PR** — keeps PRs easy to review; split further if a ticket would exceed this

If the plan already lays out phases or explicit ticket breakpoints, match that structure rather than re-slicing.

### Step 3: Ask About Issue Type, Component, and Other Fields

Before creating any tickets, ask the user:

1. **Issue type** — default to **Story** when the work involves user-facing changes; otherwise propose an alternative (Task, Bug, etc.) and confirm with the user.
2. **Component** — always ask explicitly: *"Which component should these tickets be tagged with (e.g. Mobile, Web / Server)?"* Never assume; the project may have several components.
3. **Other custom fields** — ask whether the user wants to set any additional fields (labels, sprint, priority, etc.). Default to none.

### Step 4: Confirm Tickets

Present the complete list of tickets to the user (summary + brief overview + acceptance criteria for each). Confirm before creating, or iterate with additional input.

Summary formatting requirements:
- **Title case** for every summary. Capitalize all major words; lowercase short articles, conjunctions, and prepositions (`a`, `an`, `the`, `and`, `or`, `to`, `of`, `for`, `in`, `on`, `at`, `by`, `with`, `from`).

### Step 5: Create Each Ticket

Use `mcp__claude_ai_Atlassian__createJiraIssue` for each ticket with:

- **Summary**: Title case, action-oriented
- **Description**: Follow the template below; use `contentFormat: "markdown"`
- **Issue type**: Per Step 3 (default Story for user-facing work)
- **Status**: Backlog (most projects' default initial state for new issues)
- **Parent**: Same as the parent of the ticket mentioned in the plan (no parent if the plan ticket itself is top-level)
- **Component**: Per Step 3. If the create call doesn't accept `components` directly, set it immediately afterward via `mcp__claude_ai_Atlassian__editJiraIssue` with `{"components": [{"name": "<name>"}]}`.

### Step 6: Wrap Up

After all tickets are created, do these in order:

**6a — Close the PR associated with the original plan.**

Ask: *"Would you like to close the PR associated with the original plan?"* If you don't know which PR it is, ask the user for the link.

If yes:
1. Post a comment on the PR listing every new ticket (key + summary + link), prefixed with `[<AI model name>]` — use the currently running model's name (e.g. `[Claude Code AI]`, `[Claude Opus 4.7]`). Fall back to `[AI]` if no model name is known.
2. Close the PR (e.g. `gh pr close <number> --repo <owner>/<repo>`).

**6b — Move the original Jira ticket to Done and link it to the new tickets.**

Ask: *"Would you like to move the original Jira ticket to Done?"* If you don't know which ticket it is, ask the user for the link.

If yes:
1. Post a comment on the original ticket listing every new ticket (key + summary + link), prefixed with `[<AI model name>]` using the same convention as 6a.
2. Transition the ticket to **Done** via `mcp__claude_ai_Atlassian__transitionJiraIssue` (use `getTransitionsForJiraIssue` to find the matching transition ID).
3. Link the original ticket to every new ticket using `mcp__claude_ai_Atlassian__createIssueLink` with type **Relates** — one call per new ticket. Do this even if the user declined the Done transition; the linking is independent.

## Ticket Description Template

Each ticket description is plain text — no `## Overview` or `## Implementation` headers. Format:

```markdown
[1-3 sentences describing what this ticket accomplishes and why]

See plan: [<section name>](<github permalink with line range>)

**Acceptance Criteria:**

- [Key outcome that signals this ticket is done]
- [Another outcome]
```

Formatting rules (each one matters — Jira renders content differently from GitHub):

- **No headers.** The overview text is the first paragraph. There is no `## Overview` and no `## Implementation` section.
- **Link to the plan at the end of the overview.** Use a GitHub permalink with a `#L<start>-L<end>` line-range anchor pointing to the relevant phase/section in the plan. Never copy the implementation details inline — the plan is the source of truth.
- **`**Acceptance Criteria:**` is plain bolded inline text**, not a header. Jira renders it as bold text above the bullet list.
- **Acceptance criteria are plain bullets** (`-` or `*`), NOT markdown checklists (`- [ ]`). Jira does not render markdown checklists.
- Use `contentFormat: "markdown"` on every create/edit call.
