---
name: gdt-notion
description: >
  Backlog manager for the gdt-syv Godot project. Handles Notion CRUD for the Godot-specific
  10-field backlog schema (Task, Status, Priority, Category, Scope, Effort, Description,
  Dependencies, Nodes/Scenes, Started). Also acts as a proactive agent — suggests adding
  items to the backlog when decisions, bugs, features, or scope changes arise during work.
  Triggers: "notion", "backlog", "add task", "show backlog", "query backlog", "update entry",
  "agrega tarea", "muestra el backlog", "agregar al backlog".
user_invocable: true
---

# gdt-notion — SyV Backlog Manager

All operations use `mcp__notion__*` tools. Fall back to curl only when MCP can't handle it (database creation, bulk ops >= 5 items).

---

## Fixed IDs

| Resource | ID |
|----------|-----|
| Automatic (root) | `30ebaef9-c70d-80a6-b26e-c08ed3cdc3a1` |
| gdt-syv page | `33fbaef9-c70d-8158-87d9-e25dc27a94db` |
| Backlog database | `33fbaef9-c70d-81a9-bd7f-d1ff24fe9f5c` |

---

## Backlog Schema (LOCKED — exactly 10 fields)

| # | Field | Notion Type | Allowed values / format |
|---|-------|-------------|------------------------|
| 1 | `Task` | title | Short imperative: "Implement hex grid", "Fix unit movement", "Add fog-of-war" |
| 2 | `Status` | select | `Pending` · `In Progress` · `Blocked` · `Done` · `Cut` |
| 3 | `Priority` | select | `Critical` · `High` · `Medium` · `Low` · `Nice-to-have` |
| 4 | `Category` | select | `Core Mechanic` · `UI/UX` · `Networking` · `Content` · `Audio` · `Visual` · `AI` · `Tools` · `Bug` · `Polish` |
| 5 | `Scope` | select | `Prototype` · `Alpha` · `Beta` · `Release` |
| 6 | `Effort` | select | `Trivial` · `Small` · `Medium` · `Large` · `Epic` |
| 7 | `Description` | rich_text | What it is and why it matters |
| 8 | `Dependencies` | rich_text | System dependencies: "Requires hex grid movement", "After protocol layer" |
| 9 | `Nodes/Scenes` | rich_text | Godot paths: `HexGrid.tscn, Unit.gd, server/turn_manager.gd` |
| 10 | `Started` | date | ISO date (`YYYY-MM-DD`) set when work begins |

### Schema rules

- **Never add, rename, or remove fields.** The schema is locked at 10 fields.
- If the user wants extra info, use `Description` or `Dependencies`.
- `last_edited_time` is automatic — never set it.

### Status semantics

| Status | Meaning |
|--------|---------|
| `Pending` | Not started |
| `In Progress` | Actively being worked on |
| `Blocked` | Waiting on another system or decision |
| `Done` | Completed and verified |
| `Cut` | Intentionally removed from scope (normal in game dev, not a failure) |

### Category → SyV architecture mapping

| Category | Maps to |
|----------|---------|
| Core Mechanic | `shared/` — hex grid, turn phases, unit logic |
| UI/UX | `client/` — HUD, menus, input handling |
| Networking | `protocol/`, transport layer, ENet/Steam Sockets |
| Content | Assets, maps, unit definitions, data files |
| Audio | Sound effects, music, audio buses |
| Visual | Shaders, VFX, animations, camera |
| AI | Bot logic, pathfinding, decision-making |
| Tools | Editor plugins, debug tools, dev utilities |
| Bug | Defects in any system |
| Polish | Juice, feel, minor UX improvements |

### Effort → AI Model mapping

| Effort | Claude | Description |
|--------|--------|-------------|
| `Trivial` | Haiku | Typos, renames, single-line fixes, config tweaks |
| `Small` | Haiku | Small self-contained changes, simple signal wiring |
| `Medium` | Sonnet | Multi-file refactors, simple features, test suites |
| `Large` | Opus | Complex features, multi-system integration, networking |
| `Epic` | Opus | Architecture changes, full subsystem rewrites |

### Scope semantics (SyV milestones)

| Scope | What belongs here |
|-------|-------------------|
| `Prototype` | Core loop proof: hex grid + turns + basic networking |
| `Alpha` | All core mechanics playable, placeholder art OK |
| `Beta` | Content-complete, polish pass, real assets |
| `Release` | Ship-quality, performance, packaging |

---

## Proactive Backlog Agent Behavior

**This is always active, not just when the skill is explicitly invoked.**

During any work on this project, when any of the following occur:

- A **decision** is made (architecture, convention, approach)
- A **bug** is found or fixed
- A **feature** is discussed or requested
- A **scope change** happens (something gets cut or added)
- A **blocked dependency** is identified
- A **TODO/FIXME** is found in code
- The user says something like "we should...", "eventually we need...", "that's broken"

**Ask the user before adding:**

- In Spanish context: *"Deberia poner esto en el Backlog?"*
- In English context: *"Should I add this to the Backlog?"*

Match the language the user is currently using. Never silently add items — always ask first with a brief proposed entry:

```
Deberia poner esto en el Backlog?
  Task: "Implement fog-of-war per scope"
  Category: Core Mechanic | Priority: High | Scope: Prototype | Effort: Large
```

If the user confirms ("dale", "si", "yes", "go", "do it"), add it immediately. If they adjust ("but make it Medium"), apply the adjustment.

---

## MCP Tool Reference

### Query the backlog (list entries)

```
mcp__notion__API-query-data-source
  data_source_id: "33fbaef9-c70d-81a9-bd7f-d1ff24fe9f5c"
  page_size: 100
  filter: { ... }       # optional
  sorts: [{"property": "Priority", "direction": "ascending"}]  # optional
```

Common filters:
```json
// All open items
{"and": [
  {"property": "Status", "select": {"does_not_equal": "Done"}},
  {"property": "Status", "select": {"does_not_equal": "Cut"}}
]}

// By category
{"property": "Category", "select": {"equals": "Networking"}}

// By scope
{"property": "Scope", "select": {"equals": "Prototype"}}

// Critical/High only
{"or": [
  {"property": "Priority", "select": {"equals": "Critical"}},
  {"property": "Priority", "select": {"equals": "High"}}
]}

// Blocked items
{"property": "Status", "select": {"equals": "Blocked"}}
```

Handle `has_more` / `next_cursor` for pagination.

### Display format

Show backlog entries as a compact table:

```
## SyV Backlog — Open Items (N)

| # | Task | Status | Priority | Category | Scope | Effort |
|---|------|--------|----------|----------|-------|--------|
| 1 | Implement hex grid | In Progress | Critical | Core Mechanic | Prototype | Large |
| 2 | Add unit movement | Pending | High | Core Mechanic | Prototype | Medium |
```

For detailed view (single item), also show Description, Dependencies, Nodes/Scenes.

### Add entry to the backlog

```
mcp__notion__API-post-page
  parent: {"database_id": "33fbaef9-c70d-81a9-bd7f-d1ff24fe9f5c"}
  properties: {
    "Task":         {"title": [{"type": "text", "text": {"content": "Implement hex grid rendering"}}]},
    "Status":       {"select": {"name": "Pending"}},
    "Priority":     {"select": {"name": "Critical"}},
    "Category":     {"select": {"name": "Core Mechanic"}},
    "Scope":        {"select": {"name": "Prototype"}},
    "Effort":       {"select": {"name": "Large"}},
    "Description":  {"rich_text": [{"type": "text", "text": {"content": "Render hex grid with multi-level support. Foundation for all gameplay."}}]},
    "Dependencies": {"rich_text": [{"type": "text", "text": {"content": ""}}]},
    "Nodes/Scenes": {"rich_text": [{"type": "text", "text": {"content": "shared/hex_grid.gd, client/hex_grid_renderer.tscn"}}]},
    "Started":      {"date": {"start": "2026-04-11"}}
  }
```

- Only set `Started` if work is actually beginning. Omit for new pending tasks.
- `Status` defaults to `Pending` for new items.
- Always infer `Effort` based on complexity rules above — never ask the user.
- Always infer `Category` from the nature of the task — ask only if ambiguous.

### Update an entry

```
mcp__notion__API-patch-page
  page_id: "ENTRY_PAGE_ID"
  properties: {
    "Status": {"select": {"name": "Done"}}
  }
```

Only send fields being changed.

### Add a comment to an entry

```
mcp__notion__API-create-a-comment
  parent: {"page_id": "ENTRY_PAGE_ID"}
  rich_text: [{"type": "text", "text": {"content": "Completed: implemented hex grid with axial coordinates"}}]
```

Comment every status change with what was done or why it was blocked/cut.

### Bulk operations (>= 5 items)

For 5+ updates, use Python via Bash:

```python
import os, requests

TOKEN = os.environ["NOTION_API_TOKEN"]
HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Notion-Version": "2022-06-28",
    "Content-Type": "application/json",
}
DB_ID = "33fbaef9-c70d-81a9-bd7f-d1ff24fe9f5c"

def update_page(page_id: str, props: dict) -> bool:
    r = requests.patch(f"https://api.notion.com/v1/pages/{page_id}", headers=HEADERS, json={"properties": props})
    return r.status_code == 200

def query_db() -> list:
    results, cursor = [], None
    while True:
        body = {"page_size": 100}
        if cursor:
            body["start_cursor"] = cursor
        r = requests.post(f"https://api.notion.com/v1/databases/{DB_ID}/query", headers=HEADERS, json=body)
        data = r.json()
        results += data.get("results", [])
        if not data.get("has_more"):
            break
        cursor = data["next_cursor"]
    return results
```

Rule: <= 4 items -> MCP tools. >= 5 items -> Python script.

---

## Behavior Rules

1. **MCP first**: Always use `mcp__notion__*` tools. Curl/Python only for bulk ops or MCP limitations.
2. **Schema locked**: Exactly 10 fields. Never add, rename, or remove. Refuse requests to add custom fields.
3. **Effort always inferred**: Set effort based on complexity rules. Never ask the user for effort.
4. **Category always inferred**: Set category based on what system the task touches. Ask only if genuinely ambiguous.
5. **Scope always set**: Infer from project phase and task nature. Default to current milestone if unclear.
6. **Confirmation before adding**: Always ask the user before creating a new backlog entry. Show the proposed entry compactly.
7. **Confirmation for deletes**: Required before any delete operation.
8. **Comment every status change**: Workers and humans must add a comment alongside every status update.
9. **Token safety**: Never print, log, or expose `NOTION_API_TOKEN`.
10. **Concise output**: Short responses with status counts. No verbose JSON dumps.
11. **Pagination**: Always handle `has_more` / `next_cursor`.
12. **Field language**: All backlog field values are in English. Agent conversation follows user's language.
