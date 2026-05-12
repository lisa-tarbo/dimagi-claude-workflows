# USS Code-Review Specialist Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `/uss-review` command to the `uss-tech` plugin that runs code-review's 5 standard reviewers in parallel with a new USS-specific 6th reviewer, then synthesizes results in two blocks (standard code-review output, then a USS section with audience-bucketed user-facing changes).

**Architecture:** New specialist agent at `plugins/uss-tech/agents/uss-reviewer.md` follows code-review's agent template. New orchestrator at `plugins/uss-tech/commands/uss-review.md` spawns all 6 agents in parallel (sibling-path lookup for code-review's 5), collects JSON outputs, renders standard code-review synthesis followed by a USS section with `U`-prefixed findings and `User-Facing Changes by Audience`. The expand/fix loop is owned by `/uss-review` and accepts both number spaces.

**Tech Stack:** Claude Code plugin format (Markdown + YAML frontmatter). Cross-plugin reference via `${CLAUDE_PLUGIN_ROOT}/../code-review/agents/`. Agents emit JSON-per-dimension into a shared temp dir, same contract used by code-review today.

**Spec:** `docs/superpowers/specs/2026-05-12-uss-reviewer-design.md`

---

## Task 1: USS reviewer agent file

Creates the new specialist agent. Mirrors the structure of `plugins/code-review/agents/maintainability-reviewer.md` but with USS-specific focus areas and the extended JSON schema (`user_facing_changes` field).

**Files:**
- Create: `plugins/uss-tech/agents/uss-reviewer.md`

- [ ] **Step 1: Write the agent file**

Create `plugins/uss-tech/agents/uss-reviewer.md` with the following content (use the Write tool):

````markdown
# USS Impact Reviewer Agent

You are a specialist code reviewer focused exclusively on **USS Tech impact**: how the changes in this code affect different audiences on the shared CommCare platform. USS Tech is a Dimagi sub-team building on a platform also used by SaaS customers. Your job is to make the audience and impact of each change clear — not to flag the absence of feature-flag gating as a defect. Ungated changes are often fine; the reviewer's role is to surface what's affected so the author can confirm the impact is intentional.

## Your Inputs

You receive in your prompt:
- **Code location**: paths to files/directories to read
- **Language/framework**: the tech stack (typically Python/Django for commcare-hq)
- **Purpose**: what the code is supposed to do
- **Output path**: where to write your findings JSON

## Your Process

### Step 1: Read All the Code

Read every file in scope. As you read, build a mental model of the behavior changes the diff introduces — both the obvious user-visible ones and the subtler ones (shared-helper changes, default-value changes, log-level changes, schema migrations, signal handlers).

### Step 2: Inventory User-Facing Changes by Audience

For each behavior change in the diff, identify the audience: who actually experiences this change?

Look for gating idioms in the codebase to determine the audience for each change:
- **Feature flag checks** — conditional code paths controlled by a named flag (e.g., `if some_flag_check(domain): ...`, decorators that require a flag, configuration values that depend on a flag).
- **Subscription tier checks** — code that branches on a subscription level (e.g., checks for `Pro`, `Enterprise`, or higher tiers).
- **Project setting checks** — code conditional on a per-project setting/boolean (e.g., `if project.settings.enable_x: ...`).
- **Ungated** — code that has no audience-narrowing check and therefore affects all CommCare users on the shared platform.

When you identify a change, tag its audience using one of these string forms:
- `flag:<NAME>` — behind a feature flag
- `subscription:<NAME>+` — limited to a subscription tier or higher (e.g., `subscription:Pro+`)
- `setting:<NAME>` — gated on a project setting
- `ungated` — affects all CommCare users

If a single change appears under multiple audiences (e.g., gated by a flag AND a subscription), pick the most-specific narrowing audience. If you're not confident which gate applies, note your uncertainty in the change description.

### Step 3: Check Gate Correctness (when gating is present)

When a gate exists, verify it does what's intended. This is **not** about whether to gate — it's about whether existing gates work.

- **Right flag name?** Does the check reference the flag the author meant to reference?
- **Right scope?** Per-project vs. per-user vs. per-domain — does the scope match the intent of the change?
- **Right boolean direction?** Are `enabled` / `disabled` / negations applied correctly?
- **Bypass paths?** Are all entry points to the gated behavior covered, or is there an ungated path that reaches the same code (e.g., an API endpoint, a management command, a background task)?

Findings here are real bugs — the gate is present but wrong.

### Step 4: Surface Blast Radius for Ungated Changes

For changes you've tagged `ungated`, check these specific places where impact may be wider than the author expects:

- **Database migrations** — schema changes apply to all projects, regardless of any flag.
- **Signal handlers, signals, receivers** — fire for every triggering event unless explicitly gated inside the handler.
- **Background tasks, Celery tasks, scheduled jobs** — run on schedule for all data unless they self-gate.
- **Shared caches, queues, counters** — load or contention from USS work affects all platform users.
- **Module-level side effects, new imports** — execute at startup for the whole platform; can affect import time, memory, or pull in new dependencies.
- **Changes to shared utilities, helpers, base classes** — every caller is affected, not just the USS feature that prompted the change.
- **Default-value changes** — if the new default propagates to existing ungated callers, they all behave differently now.
- **Performance** — slow queries on shared tables, lock contention on shared rows, N+1 patterns hit at scale across all tenants.

For each blast-radius concern, write a finding asking the author to confirm the impact is intentional. The finding is for surfacing, not for compliance.

### Step 5: Write Your Findings

Write a JSON file to the output path with the schema defined in Output Format below.

Findings are bounded to gate-correctness bugs (Step 3) and blast-radius concerns (Step 4). The user-facing changes inventory (Step 2) is its own field — not a list of findings.

Do **not** create findings to flag the absence of a gate on a change that has bounded impact. An ungated change with no blast-radius concern is fine — it appears in the `ungated` bucket of `user_facing_changes` for transparency, but is not a finding.

## Output Format

Write a JSON file to the output path:

```json
{
  "dimension": "uss-impact",
  "summary": "2-3 sentence assessment of overall audience clarity and any notable blast-radius risks. Be neutral on whether changes are gated.",
  "findings": [
    {
      "severity": "critical|major|minor|suggestion",
      "title": "Short descriptive title (max 8 words)",
      "location": "path/to/file.py:L10-L25 (or 'throughout' if widespread)",
      "description": "What the gate-correctness bug or blast-radius risk is, and what could go wrong concretely.",
      "suggestion": "How to fix the gate, or how to bound the impact. For surfacing findings, what to confirm and how."
    }
  ],
  "user_facing_changes": [
    {
      "audience": "flag:RELEASE_NOTES_V2",
      "changes": [
        "Users with the flag see a new sidebar widget",
        "Edit-flow bypasses the legacy validator for these users"
      ]
    },
    {
      "audience": "ungated",
      "changes": [
        "Background job switched from DEBUG to INFO logging (affects all)"
      ]
    }
  ]
}
```

**Severity guide:**
- `critical` — an actual bug, e.g., gate inverted and will run for everyone when intended only for USS, or a destructive migration affecting all tenants.
- `major` — significant impact worth confirming, e.g., touches a shared signal handler that runs for all projects.
- `minor` — subtler audience ambiguity or blast-radius risk worth flagging.
- `suggestion` — clarity opportunities (e.g., consolidating where a gate is checked, naming the audience explicitly in a comment).

## Guidelines

- Be neutral on whether to gate. Don't write findings that say "this should be behind a flag" unless the impact analysis clearly shows the change would break SaaS users — and even then, frame it as a blast-radius concern, not a gating-compliance issue.
- Be concrete in `user_facing_changes`. "Adds a new field" is too vague; "Adds a `last_seen_at` field shown on the user profile page" is useful.
- If you can't determine the audience for a change with confidence, tag it as `ungated` and call out the uncertainty in the change description. Surfacing the ambiguity is the right move.
- Stay out of other reviewers' lanes — code quality, security, architecture, naming, and maintainability are other agents' domains. Stay focused on audience and impact.
- If everything is straightforwardly bounded and there are no blast-radius concerns, say so in the summary. A short clean report is a good outcome.
````

- [ ] **Step 2: Verify the file structure**

Run:
```bash
test -f plugins/uss-tech/agents/uss-reviewer.md && wc -l plugins/uss-tech/agents/uss-reviewer.md
grep -c "^## " plugins/uss-tech/agents/uss-reviewer.md
grep -c "^### Step " plugins/uss-tech/agents/uss-reviewer.md
```

Expected:
- File exists; line count >100
- At least 5 top-level sections (`## Your Inputs`, `## Your Process`, `## Output Format`, `## Guidelines`, plus the title is `#` so doesn't count)
- 5 process steps (Steps 1–5)

- [ ] **Step 3: Verify the JSON example is valid JSON**

Extract the JSON example block and validate it:
```bash
sed -n '/^```json$/,/^```$/p' plugins/uss-tech/agents/uss-reviewer.md | sed '1d;$d' | python3 -c 'import json,sys; json.load(sys.stdin); print("OK")'
```

Expected: `OK`

- [ ] **Step 4: Commit**

```bash
git add plugins/uss-tech/agents/uss-reviewer.md
git commit -m "$(cat <<'EOF'
Add USS impact reviewer agent

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: `/uss-review` command file

Creates the orchestrator. Closely mirrors `plugins/code-review/skills/code-review/SKILL.md`'s structure — preflight, scope, parallel spawn, collect, synthesize, expand/fix loop, cleanup — adapted for the 6th agent and the two-block output format.

**Files:**
- Create: `plugins/uss-tech/commands/uss-review.md`

- [ ] **Step 1: Write the command file**

Create `plugins/uss-tech/commands/uss-review.md` with the following content (use the Write tool):

````markdown
---
description: Thorough code review including a USS Tech impact specialist — runs code-review's 5 standard reviewers in parallel with a 6th USS-specific reviewer, then renders a two-block synthesis (standard + USS section).
---

# USS Review

Orchestrate a thorough code review by spawning **6 parallel specialist reviewers** — code-review's standard 5 plus the USS impact specialist — then synthesise findings into a two-block output: the standard code-review synthesis, followed by a USS section with audience-bucketed user-facing changes.

This command depends on the `code-review` plugin being installed (it reuses code-review's 5 agent files via sibling-path lookup).

---

## Step 0: Preflight — verify code-review is installed

Resolve `${CLAUDE_PLUGIN_ROOT}` to its absolute path. The 5 code-review agent files are expected at:

```
${CLAUDE_PLUGIN_ROOT}/../code-review/agents/design-reviewer.md
${CLAUDE_PLUGIN_ROOT}/../code-review/agents/quality-reviewer.md
${CLAUDE_PLUGIN_ROOT}/../code-review/agents/smells-reviewer.md
${CLAUDE_PLUGIN_ROOT}/../code-review/agents/security-reviewer.md
${CLAUDE_PLUGIN_ROOT}/../code-review/agents/maintainability-reviewer.md
```

Verify each file exists. If any are missing, print:

> The `/uss-review` command depends on the `code-review` plugin, which does not appear to be installed. Install it via `/plugins` (add the `dimagi-claude-workflows` marketplace if needed) and retry.

Then exit without spawning any agents.

---

## Step 1: Understand scope and context

Before spawning reviewers, establish:

- **What to review** — files, directory, PR diff, or pasted code
- **Language and framework** — infer from context; confirm if ambiguous
- **Purpose of the code** — what is it trying to do?
- **Review depth** — quick pass vs. deep dive? (default: thorough)

If the user just pastes code or says "uss-review this", infer context and proceed immediately. Only ask if something critical is missing.

**Read the code first.** Use `Read` to read files and `Glob` to scan directory structure before spawning agents.

---

## Step 2: Spawn 6 parallel reviewer agents

Create a temp working directory:

```
/tmp/uss-review-{timestamp}/
```

Spawn **all 6 agents simultaneously** (in parallel, not sequentially). Each agent:
- Reads the same code (provide paths or content)
- Focuses on exactly one dimension
- Writes findings to its own JSON file in the working directory

| Agent file | Output file | Focus |
|---|---|---|
| `${CLAUDE_PLUGIN_ROOT}/../code-review/agents/design-reviewer.md` | `design.json` | Architecture, separation of concerns, coupling, layering |
| `${CLAUDE_PLUGIN_ROOT}/../code-review/agents/quality-reviewer.md` | `quality.json` | Clean code, naming, DRY, refactoring opportunities |
| `${CLAUDE_PLUGIN_ROOT}/../code-review/agents/smells-reviewer.md` | `smells.json` | Code smells, hacks, workarounds, anti-patterns |
| `${CLAUDE_PLUGIN_ROOT}/../code-review/agents/security-reviewer.md` | `security.json` | Vulnerabilities, input validation, auth, secrets, exposure |
| `${CLAUDE_PLUGIN_ROOT}/../code-review/agents/maintainability-reviewer.md` | `maintainability.json` | Testability, error handling, dead code, documentation |
| `${CLAUDE_PLUGIN_ROOT}/agents/uss-reviewer.md` | `uss.json` | USS Tech audience inventory, gate correctness, blast radius |

**Prompt to give each agent** (replace bracketed values with absolute paths and concrete values before sending):

```
Read your reviewer instructions from [/absolute/path/to/agent-file.md].

Also consult [/absolute/path/to/code-review/skills/code-review/references/language-notes.md]
for the relevant [language/framework] section — it contains idiomatic patterns,
common pitfalls, and framework conventions to check.

Then review the following code:
- Code location: [path(s) to files]
- Language/framework: [detected]
- Purpose: [brief description of what the code does]
- Output path: /tmp/uss-review-{timestamp}/[output-file]

Focus only on your assigned dimension. Be thorough within your domain.
```

The USS reviewer does **not** need to consult `language-notes.md` — its focus is platform-impact, not language idioms. Omit that line from its prompt.

---

## Step 3: Wait and collect results

Once all 6 agents complete, read all 6 JSON files from the working directory.

---

## Step 4: Synthesise findings

Process the standard 5 results the same way code-review does today:

**Deduplicate**: Multiple agents may flag the same issue from different angles. Merge into a single finding.

**Calibrate severity**: If 3+ agents flag the same root cause, that's a strong signal it's major or critical.

**Find cross-cutting themes**: Look for a single root cause that explains multiple findings.

**Assess the overall picture**: Is this code fundamentally healthy with some rough edges, or is there a deeper structural problem?

The USS reviewer's findings are kept **separate** from the standard 5 — they are not merged into the same triage table. The USS section renders independently.

---

## Step 5a: Write Block 1 — standard code-review triage

Output the standard code-review triage format (same as the `code-review` skill produces today):

### Summary
2–4 sentences on the overall state of the code. Be honest and direct. Praise what's genuinely good.

### Findings

Compact numbered table — one row per finding, no detail yet:

| #  | Sev | Finding | Location |
|----|-----|---------|----------|
| 1  | 🔴  | [title] | [file:line] |
| 2  | 🟠  | [title] | [file:line] |

Severity key: 🔴 Critical · 🟠 Major · 🟡 Minor · 💡 Suggestion

Order by severity, number sequentially starting at 1.

### 🔵 Design Observations
Higher-level architectural concerns spanning multiple findings.

### ✅ What's Working Well
2–4 things done well.

---

## Step 5b: Write Block 2 — USS section

After Block 1, output a `---` separator and then:

```
## USS Impact

### Summary
[2–3 sentences from uss.json summary field]

### USS Findings
| #  | Sev | Finding | Location |
|----|-----|---------|----------|
| U1 | 🔴  | [title] | [file:line] |
| U2 | 🟠  | [title] | [file:line] |

### User-Facing Changes by Audience

**Ungated (affects all CommCare users)**
- [change 1]
- [change 2]

**Behind flag `FLAG_NAME`**
- [change]

**Subscription `Pro+`**
- [change]

**Project setting `setting_name`**
- [change]
```

**Numbering**: USS findings use `U` prefix (`U1`, `U2`, ...). Order by severity, number sequentially starting at `U1`.

**Audience ordering** in `User-Facing Changes by Audience`:
1. `ungated` bucket first (highest-leverage information — widest impact)
2. `flag:*` buckets next
3. `subscription:*` buckets next
4. `setting:*` buckets last

Within each bucket, list changes in the order they appear in the USS reviewer's JSON output.

**Omit empty buckets entirely** — do not render a heading for an audience with zero changes.

If the USS reviewer produced no findings at all, render the section with just the Summary and the User-Facing Changes (omit the findings table). If there are no user-facing changes either, render only the Summary.

---

## Step 5c: Gate on user selection

After outputting both blocks, use `AskUserQuestion` to ask:

> "Which findings do you want to expand? Enter numbers (e.g. `1,3` or `U1,U2`), a range (`1-4`, `U1-U3`), `all`, or `fix 2,U1` to implement directly. Or just ask about any finding."

---

## Step 5d: Expand and fix loop

Parse the user's response from Step 5c. The parser recognises both the main numbering (`1`, `2`, ...) and the USS prefix (`U1`, `U2`, ...):

| Input pattern | Action |
|---------------|--------|
| `1,3,5` | Expand main findings 1, 3, 5 |
| `U1,U3` | Expand USS findings U1, U3 |
| `1-4` | Expand main findings 1 through 4 |
| `U1-U3` | Expand USS findings U1 through U3 |
| `1,U2` | Mixed list — expand main #1 and USS #2 |
| `all` | Expand all findings in both tables |
| `fix 2,U1` | Expand + implement main #2 and USS #1 |
| `fix all` | Expand + implement all findings |
| Natural language (e.g. "tell me more about the SQL issue") | Map to the closest matching finding by title/topic; if no confident match, ask the user to clarify which finding they mean |

**Expanding a finding** means writing the full detail block:

```
**[emoji] Title** (`path/to/file.py`, line X–Y)

What the problem is and why it matters — the actual consequence or risk.

*Suggestion:* What to do instead, with a brief code snippet if it genuinely helps.
```

Use the severity guide from Step 5a.

**Implementing a finding** (`fix N` or `fix UN`): after expanding, make the code change immediately. Summarise what was changed in 1–2 sentences.

**After expanding/fixing**, use `AskUserQuestion` to ask:

> "Anything else to address? Pick more numbers (`1`, `U2`, ...), ask about a finding, or say 'done'."

If the user pushes back on a finding ("I disagree with U2"), engage with their reasoning directly — they may have context that changes the assessment. This is a conversation, not a report.

Exit the loop when the user says "done" or when their response cannot be interpreted as a finding selection, fix request, or pushback.

---

## Step 6: Cleanup

Remove the temp working directory:
```bash
rm -rf /tmp/uss-review-{timestamp}/
```

---

## Tone and style

- Write like a respected senior colleague, not a linter
- Explain the *why* behind every non-trivial finding
- Avoid piling on for the same root issue — note the pattern once
- Adapt depth to context: a 20-line utility vs. a 500-line module warrant different depth
- In the USS section, frame findings as confirmation prompts rather than compliance failures. Ungated work is not, by itself, a problem.

---

## Fallback: no sub-agents available

If running in an environment without sub-agent support, review the code by working through each dimension in sequence using the 6 agent files as checklists. The output format is identical (two blocks).
````

- [ ] **Step 2: Verify the command file structure**

Run:
```bash
test -f plugins/uss-tech/commands/uss-review.md && wc -l plugins/uss-tech/commands/uss-review.md
grep -c "^## Step " plugins/uss-tech/commands/uss-review.md
grep -c "code-review/agents/" plugins/uss-tech/commands/uss-review.md
```

Expected:
- File exists, line count >150
- Step 0–6 + sub-steps (5a, 5b, 5c, 5d) → 10 step headings (5 numbered + 4 lettered + 1 fallback "Step" if applicable; main `^## Step ` count ≥ 7)
- At least 6 references to `code-review/agents/` (one per code-review agent in the agent table, plus possibly more in Step 0)

- [ ] **Step 3: Verify YAML frontmatter parses**

```bash
python3 -c "
import yaml
with open('plugins/uss-tech/commands/uss-review.md') as f:
    content = f.read()
parts = content.split('---', 2)
assert len(parts) >= 3, 'Missing frontmatter'
fm = yaml.safe_load(parts[1])
assert 'description' in fm, 'Missing description field'
print('OK -', fm['description'][:60])
"
```

Expected: `OK - Thorough code review including a USS Tech impact specialist…`

- [ ] **Step 4: Commit**

```bash
git add plugins/uss-tech/commands/uss-review.md
git commit -m "$(cat <<'EOF'
Add /uss-review command orchestrating standard + USS reviewers

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Update uss-tech plugin README

Document the new command and the `code-review` dependency. Small, focused diff — keeps the README's prose intact and appends a section.

**Files:**
- Modify: `plugins/uss-tech/README.md`

- [ ] **Step 1: Add a Commands section to the plugin README**

Append the following to `plugins/uss-tech/README.md` (use the Edit tool to insert before EOF):

```markdown

## Commands

- `/uss-review` — Thorough code review with a USS impact specialist. Runs
  the 5 standard reviewers from the `code-review` plugin in parallel with
  a USS-specific 6th reviewer, then renders the standard code-review
  synthesis followed by a USS section with audience-bucketed user-facing
  changes.

  **Requires the `code-review` plugin** to also be installed.
```

- [ ] **Step 2: Verify the addition**

```bash
grep -c "^## Commands$" plugins/uss-tech/README.md
grep "/uss-review" plugins/uss-tech/README.md
```

Expected:
- `1`
- A line mentioning `/uss-review`

- [ ] **Step 3: Commit**

```bash
git add plugins/uss-tech/README.md
git commit -m "$(cat <<'EOF'
Document /uss-review command in uss-tech README

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Update root plugins README and bump marketplace version

Surface the new command in the marketplace listing so users browsing `/plugins` see it. Bump uss-tech's version since it's gaining a user-facing surface.

**Files:**
- Modify: `plugins/README.md`
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Locate the uss-tech entry in the root plugins README**

```bash
grep -n "^## uss-tech" plugins/README.md
```

Expected: a single match with a line number.

- [ ] **Step 2: Read the uss-tech section to see its existing structure**

```bash
sed -n '/^## uss-tech/,/^## /p' plugins/README.md
```

This shows the current entry. It probably lists skills (`jira-project-management`) but no commands. We will add a Commands subsection alongside.

- [ ] **Step 3: Add the command entry**

Use the Edit tool to add a `**Commands**` block to the uss-tech section in `plugins/README.md`. Insert it ABOVE the existing `**Skills**` block (following the convention used in `dev-utils` and `code-review` sections of the same file):

```markdown
**Commands**

- `/uss-review`: Thorough code review with a USS impact specialist. Runs code-review's 5 reviewers in parallel with a USS-specific 6th reviewer; renders the standard synthesis followed by a USS section with audience-bucketed user-facing changes. Requires the `code-review` plugin.

```

(Note the trailing blank line — preserves the section's existing whitespace.)

- [ ] **Step 4: Verify the addition**

```bash
sed -n '/^## uss-tech/,/^## [a-z]/p' plugins/README.md | grep -c "/uss-review"
```

Expected: at least `1`.

- [ ] **Step 5: Bump uss-tech version in marketplace.json**

Use the Edit tool on `.claude-plugin/marketplace.json` — change uss-tech's `"version": "0.1.0"` to `"version": "0.2.0"`. The relevant block currently reads:

```json
    {
      "name": "uss-tech",
      "source": "./plugins/uss-tech",
      "version": "0.1.0",
      "description": "Skills for the USS Tech Team"
    },
```

Change to:

```json
    {
      "name": "uss-tech",
      "source": "./plugins/uss-tech",
      "version": "0.2.0",
      "description": "Skills and commands for the USS Tech Team"
    },
```

(Description updated to reflect that the plugin now provides commands as well as skills.)

- [ ] **Step 6: Verify marketplace.json is valid JSON**

```bash
python3 -c "import json; data = json.load(open('.claude-plugin/marketplace.json')); uss = [p for p in data['plugins'] if p['name'] == 'uss-tech'][0]; assert uss['version'] == '0.2.0', uss; print('OK -', uss)"
```

Expected: `OK - {'name': 'uss-tech', ..., 'version': '0.2.0', ...}`

- [ ] **Step 7: Commit**

```bash
git add plugins/README.md .claude-plugin/marketplace.json
git commit -m "$(cat <<'EOF'
Surface /uss-review in plugins index and bump uss-tech to 0.2.0

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Manual smoke test

Plugin code is interpreted by Claude at runtime — there's no automated test loop that covers end-to-end command behavior. This task verifies the implementation works on a real piece of code before declaring the feature done.

**Files:** none modified in this task (unless issues are found).

- [ ] **Step 1: Pick a target for review**

Ask the user: "What code should we run `/uss-review` against for a smoke test? Options:

1. The most recent commit on `es/uss-tech` (small and self-referential — fine for shape, not for USS behavior)
2. A real commcare-hq PR or branch the user has handy
3. A file the user nominates"

Wait for the user's choice.

- [ ] **Step 2: Run /uss-review**

Invoke the command on the chosen target. Observe:

- Does Step 0 preflight succeed (or fail with the expected error if `code-review` is missing)?
- Do all 6 agents spawn in parallel?
- Do all 6 JSON files appear in `/tmp/uss-review-{timestamp}/`?
- Is Block 1 produced in the standard code-review format?
- Is Block 2 (USS) appended after a `---` separator?
- Are USS findings `U`-prefixed?
- Is `User-Facing Changes by Audience` grouped with `ungated` first?
- Does the expand/fix loop accept `U1`, `U1-U2`, `fix U1`, and mixed inputs?

- [ ] **Step 3: Iterate on agent prompt content if needed**

If the USS reviewer's output is poorly calibrated (e.g., misses obvious audiences, flags ungated work as defective, miscategorises gates), revise `plugins/uss-tech/agents/uss-reviewer.md` and re-run. Commit any prompt refinements as separate small commits with messages like `Refine USS reviewer: <what changed>`.

- [ ] **Step 4: Declare done**

Once a real smoke test produces the expected output shape with reasonable substance, the feature is shipped on this branch. Open a PR if desired (the user's `/create-pr` command is available in `dev-utils`).

---

## Self-review notes

The plan was self-reviewed against the spec:

- **Spec coverage:** Every section of `2026-05-12-uss-reviewer-design.md` maps to a task — agent (Task 1), command/orchestration (Task 2), README (Task 3), marketplace surface + version (Task 4), smoke test (Task 5). The Open Question on cross-plugin reference is settled in Task 2 (sibling path).
- **Placeholder scan:** No `TBD` / `TODO` / `fill in` / "similar to Task N" patterns; agent and command file content are spelled out in full inside the steps.
- **Type consistency:** The audience tag forms (`flag:`, `subscription:+`, `setting:`, `ungated`) match exactly between the agent file (Task 1) and the orchestrator's rendering rules (Task 2). The USS finding prefix (`U1`, `U2`) is consistent across both files. The temp dir convention (`/tmp/uss-review-{timestamp}/`) is used consistently in Task 2 and Task 5. JSON schema field names (`dimension`, `summary`, `findings`, `user_facing_changes`, `audience`, `changes`) match between agent output and orchestrator parsing.
