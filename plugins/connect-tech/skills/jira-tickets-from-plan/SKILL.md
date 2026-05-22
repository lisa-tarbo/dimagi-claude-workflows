---
name: jira-tickets-from-plan
description: Use when creating Jira tickets from a AI plan.
---

# Jira Tickets from AI Plan

## Overview

Reviews the plan, breaks it into logically independent tickets, and creates them in Jira. It works best if the plan is supplied as a Github link. 

## Process

### Step 1: Review the Plan

Read the full plan. Identify logical breakpoints using these criteria (in priority order):

1. **Independently deployable** — each ticket can be merged and deployed without depending on other in-progress work
2. **max ~500 lines per PR** — keeps PRs easy to review; split further if a ticket would exceed this

### Step 2: Ask About Custom Fields

Before creating any tickets, ask the user:

> "Would you like to populate additional ticket fields (e.g components)?"

- **No** → proceed to Step 3 with defaults
- **Yes** → ask for each field value one at a time before creating tickets

### Step 3: Confirm Tickets

Present the complete list of tickets to the user and confirm if we are ok to proceed to ticket creation or iterate with additional information from user. 

### Step 4: Create Each Ticket

Use `mcp__claude_ai_Atlassian__createJiraIssue` for each ticket with:

- **Summary**: Clear, action-oriented description of what the ticket targets
- **Description**: Overview of the changes
- **Status**: Backlog
- **Parent**: Keep 'parent' same as the 'parent' of ticket mentioned in plan


## Ticket Description Template

```
## Overview
[1-3 sentences describing what this ticket accomplishes and why]

## Implementation

Link to the relevant section of Github Plan if a Github link is available, otherwise copy the relevant section of implementation plan verbatim here. 

## Acceptance Criteria
- [ ] [Key outcome that signals this ticket is done]
```