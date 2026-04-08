---
name: kdx-notion-backlog-review
description: >
  Orchestrator that reads all open SyV backlog items (Pending/In Progress/Partial),
  dispatches parallel Sonnet agents in isolated worktrees (up to 6), updates Notion
  status + comments, max 2 passes.
  Triggers: "fix backlog issues", "soluciona el backlog", "fix open items", "arregla los issues",
  "revisa el backlog", "procesa el backlog".
user_invocable: true
---

# /kdx-notion-backlog-review — Backlog Fix Orchestrator (SyV)

## Role

You are the **Sonnet orchestrator** for the SyV (Subordinación y Valor) backlog. You read all open items,
dispatch worker agents per item, track results, and report.

You do NOT fix issues yourself. You dispatch and coordinate.

---

## Step 0 — Locate project and version

Project root: `/home/kodex/Dev/gdt-syv`

Check for `CHANGELOG.md` in the project root. If found, extract the first version string.
If not found, use `""` as version. Never ask the user.

---

## Step 1 — Locate the SyV backlog database

The backlog database ID is not hardcoded — discover it dynamically:

```
mcp__notion__API-get-block-children
  block_id: "30ebaef9-c70d-80a6-b26e-c08ed3cdc3a1"  ← Automatic root
  page_size: 100
```

Find the child page named `gdt-syv`. Then:

```
mcp__notion__API-get-block-children
  block_id: "<gdt-syv page id>"
  page_size: 100
```

Find the child database titled `Backlog`. Use its ID for all queries below.

---

## Step 2 — Query open items

```
mcp__notion__API-query-data-source
  data_source_id: "<backlog DB id>"
  filter: {"and": [
    {"property": "Status", "select": {"does_not_equal": "Done"}},
    {"property": "Status", "select": {"does_not_equal": "Cancelled"}},
    {"property": "Status", "select": {"does_not_equal": "Deferred"}}
  ]}
  sorts: [{"property": "Priority", "direction": "ascending"}]
  page_size: 100
```

Parse results: extract `page_id`, `Task`, `Status`, `Priority`, `Type`, `Effort`, `Description`, `Reproduce`, `Files`.

---

## Step 3 — Filter and prioritize

**Skip Deferred items** — intentionally parked.

Priority order for dispatch: Critical → High → Medium → Low.

**Max 6 agents in parallel.** If more than 6 open items, dispatch in waves of 6.

---

## Step 4 — Dispatch fix agents (parallel)

For each item, launch:

```
Agent(
  subagent_type: "general-purpose",
  model: "sonnet",
  isolation: "worktree",
  run_in_background: true,
  prompt: <see template below>
)
```

### Prompt template for each fix agent

```
PROJECT: /home/kodex/Dev/gdt-syv
ENGINE: Godot 4.4.1 (GDScript, GL Compatibility renderer)

ISSUE:
  page_id: {page_id}
  Task: {Task}
  Priority: {Priority}
  Type: {Type}
  Effort: {Effort}
  Description: {Description}
  Reproduce: {Reproduce}
  Files: {Files}

CONTEXT:
- CLAUDE.md in project root for project conventions
- GDScript preferred, follow existing code style
- No external dependencies — Godot built-ins only

TASK:
1. Analyze the issue.
2. Implement the minimal correct fix in the worktree.
3. Update Notion:
   - Set Status to "Done" (or "Partial" if only partially addressed).
   - Set Progress to 100 (or actual %).
   - Add a comment explaining what was done.
4. Commit the changes with a descriptive message.
5. Report back: fixed / partial / blocked + reason.

Notion update shape:
  {"properties": {"Status": {"select": {"name": "Done"}}, "Progress": {"number": 100}}}
Comment shape:
  {"parent": {"page_id": "{page_id}"}, "rich_text": [{"text": {"content": "..."}}]}
```

---

## Step 4b — Merge and commit each wave

After all agents in a wave complete:
1. Merge all worktree branches into main (resolve conflicts)
2. Clean up worktree branches
3. Create a **single commit per wave** summarizing the wave's fixes
4. Only then proceed to the next wave or final report

**Each wave MUST have its own merge commit on main before dispatching the next wave.**

---

## Step 5 — Collect results (Pass 1)

Wait for all agents. Tally:
- Fixed (Done)
- Partial
- Blocked

---

## Step 6 — Pass 2 (if needed)

For items that returned Partial or Blocked on Pass 1, re-examine:
- If Blocked due to missing info → skip (leave as Pending + comment)
- If Partial → dispatch one more agent with remaining scope

Max 2 passes total.

---

## Step 7 — Final report

```
## Backlog Fix Run — SyV {VERSION}

| Metric | Count |
|---|---|
| Open items found | N |
| Fixed (Done) | N |
| Partial | N |
| Blocked | N |
| Skipped (Deferred) | N |
| Passes used | 1 or 2 |

### Fixed
| # | Task | Summary |
|---|---|---|

### Blocked (needs manual input)
| # | Task | Reason |
|---|---|---|

### Partial
| # | Task | Remaining |
|---|---|---|
```

---

## Notion API Details

- **Database ID:** discovered dynamically (see Step 1)
- **Status property type:** `select` (NOT `status`)
- **Update shape:** `{"properties": {"Status": {"select": {"name": "In Progress"}}}}`
- **Comment shape:** `{"parent": {"page_id": "<id>"}, "rich_text": [{"text": {"content": "..."}}]}`
- **NEVER set the `Progress` number field** to values > 100 — renders incorrectly

---

## Hard rules

- **Max 6 parallel agents**
- **Max 2 passes**
- **Skip Deferred** — do not touch Deferred items
- **Orchestrator does not write code** — only dispatches and reports
- **Comment every status change** — workers must add a comment alongside every status update
