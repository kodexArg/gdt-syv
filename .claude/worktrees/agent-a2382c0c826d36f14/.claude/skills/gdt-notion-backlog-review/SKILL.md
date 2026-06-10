---
name: gdt-notion-backlog-review
description: >
  Orchestrator that reads all open SyV backlog items (Pending/In Progress/Blocked),
  dispatches parallel agents in isolated worktrees (up to 6), updates Notion
  status + comments, max 2 passes. Uses the Godot-specific 10-field schema.
  Triggers: "fix backlog issues", "soluciona el backlog", "fix open items", "arregla los issues",
  "revisa el backlog", "procesa el backlog", "fix backlog", "run backlog".
user_invocable: true
---

# /gdt-notion-backlog-review — Backlog Fix Orchestrator (SyV)

## Role

You are the **orchestrator** for the SyV (Subordinacion y Valor) backlog. You read all open items,
dispatch worker agents per item, track results, and report.

You do NOT fix issues yourself. You dispatch and coordinate.

---

## Fixed IDs

| Resource | ID |
|----------|-----|
| gdt-syv page | `33fbaef9-c70d-8158-87d9-e25dc27a94db` |
| Backlog database | `33fbaef9-c70d-81a9-bd7f-d1ff24fe9f5c` |

---

## Schema Reference (10 fields)

| Field | Type | Values |
|-------|------|--------|
| `Task` | title | Short imperative |
| `Status` | select | `Pending` / `In Progress` / `Blocked` / `Done` / `Cut` |
| `Priority` | select | `Critical` / `High` / `Medium` / `Low` / `Nice-to-have` |
| `Category` | select | `Core Mechanic` / `UI/UX` / `Networking` / `Content` / `Audio` / `Visual` / `AI` / `Tools` / `Bug` / `Polish` |
| `Scope` | select | `Prototype` / `Alpha` / `Beta` / `Release` |
| `Effort` | select | `Trivial` / `Small` / `Medium` / `Large` / `Epic` |
| `Description` | rich_text | What and why |
| `Dependencies` | rich_text | System dependencies |
| `Nodes/Scenes` | rich_text | Godot nodes, scenes, scripts |
| `Started` | date | ISO date |

### Effort -> Agent Model mapping

| Effort | Agent model |
|--------|-------------|
| `Trivial` | haiku |
| `Small` | haiku |
| `Medium` | sonnet |
| `Large` | opus |
| `Epic` | opus |

---

## Step 1 — Query open items

```
mcp__notion__API-query-data-source
  data_source_id: "33fbaef9-c70d-81a9-bd7f-d1ff24fe9f5c"
  filter: {"and": [
    {"property": "Status", "select": {"does_not_equal": "Done"}},
    {"property": "Status", "select": {"does_not_equal": "Cut"}}
  ]}
  sorts: [{"property": "Priority", "direction": "ascending"}]
  page_size: 100
```

Parse results: extract `page_id`, `Task`, `Status`, `Priority`, `Category`, `Scope`, `Effort`, `Description`, `Dependencies`, `Nodes/Scenes`.

---

## Step 2 — Filter and prioritize

**Skip Blocked items** on first pass — they need human input, not code fixes.

**Skip Nice-to-have** unless all higher-priority items are resolved.

Priority order for dispatch: Critical -> High -> Medium -> Low.

**Max 6 agents in parallel.** If more than 6 actionable items, dispatch in waves of 6.

---

## Step 3 — Select model per item

Use the Effort -> Agent Model mapping table above.
Each agent's `model` parameter is determined by the item's Effort field.

---

## Step 4 — Dispatch fix agents (parallel)

For each item, launch:

```
Agent(
  subagent_type: "general-purpose",
  model: <from effort mapping>,
  isolation: "worktree",
  run_in_background: true,
  prompt: <see template below>
)
```

### Prompt template for each fix agent

```
PROJECT: /home/kodex/Dev/gdt-syv
ENGINE: Godot 4.4.1 (GDScript, GL Compatibility renderer)
STRUCTURE: shared/ (logic), client/ (UI), server/ (arbiter), protocol/ (message contracts)

ISSUE:
  page_id: {page_id}
  Task: {Task}
  Priority: {Priority}
  Category: {Category}
  Scope: {Scope}
  Effort: {Effort}
  Description: {Description}
  Dependencies: {Dependencies}
  Nodes/Scenes: {Nodes_Scenes}

CONTEXT:
- Read CLAUDE.md and docs/adr/ for project conventions and architecture decisions
- GDScript only, follow existing code style
- No external dependencies — Godot built-ins only
- shared/ code must work on both client and headless server

TASK:
1. Analyze the issue. Read relevant files first.
2. Implement the minimal correct fix/feature in the worktree.
3. If the item has Dependencies, verify they are met before proceeding. If not, report as Blocked.
4. Commit the changes with a descriptive message.
5. Report back: fixed / partial / blocked + reason.

DO NOT update Notion yourself. The orchestrator handles status updates.
```

---

## Step 4b — Collect results and merge

After all agents in a wave complete:

1. Review each agent's result (fixed / partial / blocked)
2. For each **fixed** agent:
   - Merge the worktree branch into main (resolve conflicts if any)
3. For each **partial** agent:
   - Review what was done and what remains
4. For each **blocked** agent:
   - Note the blocking reason
5. Clean up worktree branches
6. Create a single merge commit per wave if multiple branches merged

---

## Step 5 — Update Notion

After merging, update each item's status in Notion:

**Fixed items:**
```
mcp__notion__API-patch-page
  page_id: "<id>"
  properties: {"Status": {"select": {"name": "Done"}}}
```
+ Comment: what was done

**Partial items:**
```
mcp__notion__API-patch-page
  page_id: "<id>"
  properties: {"Status": {"select": {"name": "In Progress"}}}
```
+ Comment: what was done, what remains

**Blocked items:**
```
mcp__notion__API-patch-page
  page_id: "<id>"
  properties: {"Status": {"select": {"name": "Blocked"}}}
```
+ Comment: why it's blocked, what it needs

**Comment shape:**
```
mcp__notion__API-create-a-comment
  parent: {"page_id": "<id>"}
  rich_text: [{"type": "text", "text": {"content": "..."}}]
```

---

## Step 6 — Pass 2 (if needed)

For items that returned **Partial** on Pass 1:
- Re-examine what remains
- Dispatch one more agent with the remaining scope + context from Pass 1

For items that returned **Blocked**:
- Leave as Blocked with a comment. Do not retry.

**Max 2 passes total.**

---

## Step 7 — Final report

```
## Backlog Fix Run — SyV

| Metric | Count |
|---|---|
| Open items found | N |
| Fixed (Done) | N |
| Partial (In Progress) | N |
| Blocked | N |
| Skipped (Nice-to-have) | N |
| Passes used | 1 or 2 |

### Fixed
| # | Task | Category | Summary |
|---|------|----------|---------|

### Blocked (needs manual input)
| # | Task | Category | Reason |
|---|------|----------|--------|

### Partial (needs more work)
| # | Task | Category | Remaining |
|---|------|----------|-----------|
```

---

## Hard rules

- **Max 6 parallel agents**
- **Max 2 passes**
- **Skip Blocked** items — they need human decisions
- **Orchestrator does not write code** — only dispatches and reports
- **Comment every status change** — add a Notion comment explaining what happened
- **Model matches Effort** — respect the effort-to-model mapping
- **Agents do NOT touch Notion** — only the orchestrator updates status and comments
