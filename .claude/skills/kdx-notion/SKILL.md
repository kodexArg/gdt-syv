---
name: kdx-notion
description: >
  Manages Notion via MCP tools. ALWAYS invoke this skill when the user mentions "notion" in any context —
  viewing, searching, creating, updating, querying, or referencing Notion pages, databases, tasks, or backlogs.
  Three areas: (1) ToDo page for quick reminders and checkboxes, (2) Backlogs: fixed-schema databases for
  project task tracking, (3) Notes: free-form project pages.
  Triggers: "notion", "backlog", "add task", "show my tasks", "check off", "remind me", "agrega tarea",
  "muestra mis tareas", "create backlog", "add to backlog", "query backlog", "update entry",
  "add to notion", "check notion", "notion page", "notion database".
---

# Notion Manager (MCP)

All operations use the `mcp__notion__*` tools. **Never use curl, requests, or Bash** for Notion API calls.

---

## Workspace Structure

```
Automatic (⏩ 30ebaef9-c70d-80a6-b26e-c08ed3cdc3a1)  ← shared root for all projects
├── welp-app/
│   └── Backlog (database)
├── cotton-coveris-mvp/
│   ├── Backlog (database)
│   └── Notes (page)
├── dj-west/
│   └── Backlog (database)
└── ...future projects

ToDo (221baef9-c70d-80cc-b449-ec53a0d2cbf9)  ← personal quick reminders
```

### Project naming convention

Each project is a **page** under Automatic named after the **project folder** (e.g., `cotton-coveris-mvp`, `welp-app`, `dj-west`).
- No "Backlog —" prefix. No display names. Just the folder name.
- The Backlog is a **child database** inside the project page, always titled `Backlog`.
- Other child pages (like `Notes`) are allowed — they are plain pages with paragraph blocks, not databases.

### Known fixed pages

| Name | ID | Type |
|------|-----|------|
| Automatic | `30ebaef9-c70d-80a6-b26e-c08ed3cdc3a1` | Root for all projects |
| ToDo personal | `221baef9-c70d-80cc-b449-ec53a0d2cbf9` | Page with to_do blocks |

Projects and their databases are discovered dynamically via `mcp__notion__API-get-block-children` on the Automatic page.

---

## Area 1: ToDo Page — Quick Reminders

Unstructured checkboxes. No databases.

### List tasks

Use `mcp__notion__API-get-block-children` with `block_id: "221baef9-c70d-80cc-b449-ec53a0d2cbf9"`.
Filter results where `type == "to_do"`. Display as:

```
1. ⬜ Task text
2. ✅ Completed task
```

Keep block IDs available internally but don't show them unless needed.
Handle `has_more` pagination with `start_cursor`.

### Add tasks

Use `mcp__notion__API-patch-block-children` with `block_id: "221baef9-c70d-80cc-b449-ec53a0d2cbf9"`.
Always batch multiple tasks in a single call. The `children` array accepts `paragraph` and `bulleted_list_item` block types via the MCP schema — for to_do blocks, use Bash with curl as a **fallback only** since the MCP typed schema doesn't include `to_do` blocks.

Fallback for to_do blocks:
```bash
TOKEN=$NOTION_API_TOKEN
curl -s -X PATCH "https://api.notion.com/v1/blocks/221baef9-c70d-80cc-b449-ec53a0d2cbf9/children" \
  -H "Authorization: Bearer $TOKEN" -H "Notion-Version: 2022-06-28" -H "Content-Type: application/json" \
  -d '{"children": [{"object":"block","type":"to_do","to_do":{"rich_text":[{"type":"text","text":{"content":"TASK_TEXT"}}],"checked":false}}]}'
```

### Toggle task

Use `mcp__notion__API-update-a-block` with the block's ID:
- `block_id`: the to_do block ID
- `type`: `{"to_do": {"checked": true}}` (or `false` to uncheck)

### Delete task (confirm first)

Use `mcp__notion__API-delete-a-block` with the block's ID. **Always confirm with the user before deleting.**

---

## Area 2: Backlogs — Fixed Schema

### Backlog Schema (MANDATORY — no additions, no removals)

Every backlog database has **exactly these 11 fields**. Do NOT add extra fields. Do NOT omit fields.

| # | Field | Notion Type | Allowed values / format |
|---|-------|-------------|------------------------|
| 1 | `Task` | title | Short imperative: "Fix X", "Add Y", "Refactor Z" |
| 2 | `Status` | select | `Pending` · `In Progress` · `Partial` · `Done` · `Cancelled` |
| 3 | `Priority` | select | `Critical` · `High` · `Medium` · `Low` |
| 4 | `Type` | select | `Bug` · `Feature` · `Enhancement` · `Task` · `Idea` |
| 5 | `Effort` | select | `Low` · `Medium` · `High` |
| 6 | `Progress` | number (percent) | Integer 0–100. `Done`=100, `Partial`=actual %, `Pending`/`Cancelled`=0 |
| 7 | `Description` | rich_text | What it is and why it matters |
| 8 | `Reproduce` | rich_text | BDD steps (Given/When/Then) for bugs; blank for features |
| 9 | `Files` | rich_text | Comma-separated paths relative to project root |
| 10 | `Started` | date | ISO date (`YYYY-MM-DD`) set when work begins |
| 11 | `Version` | rich_text | Semver string auto-read from `CHANGELOG.md`; **empty if no CHANGELOG.md** |

**Schema enforcement rules:**
- When creating a backlog: use exactly these 11 fields, no more.
- When adding entries: only set these 11 properties. Ignore any extra fields that may exist on legacy databases.
- When the user asks for an extra field: refuse and explain the schema is locked. Suggest using `Description` for additional context.
- `last_edited_time` is automatic — never set it.

**Version auto-detection (required on every new entry):**

1. Check if `CHANGELOG.md` exists in the current working directory (project root).
   - Use the Read tool or Glob — **do NOT use Bash `cat`**.
2. If **not found**: set `Version` to empty string (`""`). Do not ask the user.
3. If **found**: read it and extract the first version heading. Common patterns:
   - `## [1.2.3]` → extract `1.2.3`
   - `## v1.2.3` → extract `v1.2.3`
   - `# 1.2.3` → extract `1.2.3`
   - Use the first match from the top of the file (= latest release).
4. Set `Version` to the extracted string (e.g., `"1.2.3"` or `"v1.2.3"`).

### Effort → AI Model Mapping

| Effort | Claude | Gemini |
|--------|--------|--------|
| `Low` | Haiku | Flash |
| `Medium` | Sonnet | Gemini 3 Low |
| `High` | Opus | Gemini 3 Pro |

Assignment:
- **Low**: typos, renames, single-line fixes, config tweaks, CSS cosmetics
- **Medium**: refactors, multi-file bugs, simple features, test suites
- **High**: security, architecture, data migration, complex features, auth flows

---

## Area 3: Notes — Free-form Project Pages

Each project can have a `Notes` child page for unstructured content: decisions, context, meeting notes, links.

### Create a Notes page

1. Use `mcp__notion__API-post-page` with `parent: {"page_id": "PROJECT_PAGE_ID"}` and title `Notes`.
2. Add content with `mcp__notion__API-patch-block-children` using `paragraph` blocks.

Notes pages are **plain text only** — no databases, no schemas. Just paragraphs and optionally bulleted lists.

### Add a note (append paragraph)

```
mcp__notion__API-patch-block-children
  block_id: "NOTES_PAGE_ID"
  children: [{"type": "paragraph", "paragraph": {"rich_text": [{"type": "text", "text": {"content": "Note text here"}}]}}]
```

---

## Creating a New Project

When the user says "create a project for X" or "create backlog for X":

1. **Create the project page** under Automatic:
   ```
   mcp__notion__API-post-page
     parent: {"page_id": "30ebaef9-c70d-80a6-b26e-c08ed3cdc3a1"}
     properties: {"title": {"title": [{"type": "text", "text": {"content": "project-folder-name"}}]}}
   ```
   Use the project's folder name (lowercase, hyphenated). No "Backlog —" prefix.

2. **Create the Backlog database** inside it using `mcp__notion__API-create-a-data-source` with the locked 10-field schema (see "Create a backlog" in MCP Tool Reference below).

3. Optionally create a `Notes` page if the user asks.

---

## MCP Tool Reference

### Search for pages or databases

```
mcp__notion__API-post-search
  query: "search term"
  filter: {"property": "object", "value": "page"}    # or "data_source"
  page_size: 10
```

### Inspect page structure (find child databases)

```
mcp__notion__API-get-block-children
  block_id: "PAGE_ID"
  page_size: 100
```

Look for blocks where `type == "child_database"` — the `.child_database.title` gives the DB name and the `.id` is the database ID.

### Retrieve database schema

```
mcp__notion__API-retrieve-a-database
  database_id: "DB_ID"
```

Returns `.properties` with full schema. Use to verify fields before writing.

### Query a database (list entries)

```
mcp__notion__API-query-data-source
  data_source_id: "DB_ID"
  page_size: 100
  filter: { ... }       # optional
  sorts: [{"property": "Status", "direction": "ascending"}]  # optional
```

Common filters:
```json
{"property": "Status", "select": {"equals": "Pending"}}
{"property": "Priority", "select": {"equals": "Critical"}}
{"and": [
  {"property": "Status", "select": {"does_not_equal": "Done"}},
  {"property": "Status", "select": {"does_not_equal": "Cancelled"}}
]}
```

Handle `has_more` / `next_cursor` for pagination.

### Create a backlog (new database)

```
mcp__notion__API-create-a-data-source
  parent: {"page_id": "PARENT_PAGE_ID"}
  title: [{"type": "text", "text": {"content": "Backlog"}}]
  properties: {
    "Task":        {"title": {}},
    "Status":      {"select": {"options": [
      {"name": "Pending", "color": "default"},
      {"name": "In Progress", "color": "blue"},
      {"name": "Partial", "color": "yellow"},
      {"name": "Done", "color": "green"},
      {"name": "Cancelled", "color": "red"}
    ]}},
    "Priority":    {"select": {"options": [
      {"name": "Critical", "color": "red"},
      {"name": "High", "color": "orange"},
      {"name": "Medium", "color": "yellow"},
      {"name": "Low", "color": "green"}
    ]}},
    "Type":        {"select": {"options": [
      {"name": "Bug", "color": "red"},
      {"name": "Feature", "color": "blue"},
      {"name": "Enhancement", "color": "purple"},
      {"name": "Task", "color": "default"},
      {"name": "Idea", "color": "pink"}
    ]}},
    "Effort":      {"select": {"options": [
      {"name": "Low", "color": "green"},
      {"name": "Medium", "color": "yellow"},
      {"name": "High", "color": "red"}
    ]}},
    "Progress":    {"number": {"format": "percent"}},
    "Description": {"rich_text": {}},
    "Reproduce":   {"rich_text": {}},
    "Files":       {"rich_text": {}},
    "Started":     {"date": {}},
    "Version":     {"rich_text": {}}
  }
```

### Add entry to a backlog

```
mcp__notion__API-post-page
  parent: {"database_id": "DB_ID"}
  properties: {
    "Task":        {"title": [{"type": "text", "text": {"content": "Fix login timeout"}}]},
    "Status":      {"select": {"name": "Pending"}},
    "Priority":    {"select": {"name": "High"}},
    "Type":        {"select": {"name": "Bug"}},
    "Effort":      {"select": {"name": "Medium"}},
    "Progress":    {"number": 0},
    "Description": {"rich_text": [{"type": "text", "text": {"content": "Login times out after 30s on slow networks"}}]},
    "Reproduce":   {"rich_text": [{"type": "text", "text": {"content": "Given a user on 3G\\nWhen they submit login\\nThen timeout after 30s"}}]},
    "Files":       {"rich_text": [{"type": "text", "text": {"content": "backend/auth/views.py, backend/auth/timeout.py"}}]},
    "Started":     {"date": {"start": "2026-03-19"}},
    "Version":     {"rich_text": [{"type": "text", "text": {"content": "1.4.2"}}]}
  }
```

Only set `Started` if work is actually beginning. Omit it for new pending tasks.
Always run Version auto-detection (see schema section) before submitting — set `Version` to `""` if no CHANGELOG.md found.

### Update an entry

```
mcp__notion__API-patch-page
  page_id: "ENTRY_PAGE_ID"
  properties: {
    "Status":   {"select": {"name": "Done"}},
    "Progress": {"number": 100}
  }
```

Only send the fields being changed.

### Bulk operations (≥5 items)

For 5+ updates, use a Python script with the `requests` library instead of calling MCP tools one by one. This saves tokens and is faster:

```python
import os, requests

TOKEN = os.environ["NOTION_API_TOKEN"]
HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Notion-Version": "2022-06-28",
    "Content-Type": "application/json",
}

def update_page(page_id: str, props: dict) -> bool:
    r = requests.patch(f"https://api.notion.com/v1/pages/{page_id}", headers=HEADERS, json={"properties": props})
    return r.status_code == 200

def query_db(db_id: str) -> list:
    results, cursor = [], None
    while True:
        body = {"page_size": 100}
        if cursor:
            body["start_cursor"] = cursor
        r = requests.post(f"https://api.notion.com/v1/databases/{db_id}/query", headers=HEADERS, json=body)
        data = r.json()
        results += data.get("results", [])
        if not data.get("has_more"):
            break
        cursor = data["next_cursor"]
    return results
```

Rule: ≤4 items → MCP tools. ≥5 items → Python script via Bash.

---

## Behavior Rules

1. **MCP first**: Always use `mcp__notion__*` tools. Only fall back to curl/Python for to_do block creation (MCP schema limitation) or bulk ops (≥5 items).
2. **Schema locked**: Backlogs have exactly 11 fields. Never add, rename, or remove fields. Refuse requests to add custom fields.
3. **Effort always set**: Infer effort for every new entry based on the complexity rules above.
11. **Version always detected**: Before adding any entry, check for `CHANGELOG.md` in the project root. Auto-populate `Version` from the first version heading found. If no CHANGELOG.md, set empty. Never ask the user for the version.
4. **Progress consistency**: `Done`=100, `Partial`=actual %, `Pending`/`Cancelled`=0. Update Progress when changing Status.
5. **Smart matching**: If a search is ambiguous, show options and ask the user to pick.
6. **Confirmation**: Required only for delete operations.
7. **Token safety**: Never print, log, or expose `NOTION_API_TOKEN`.
8. **Concise output**: Short responses with status counts. No verbose JSON dumps.
9. **Pagination**: Always handle `has_more` / `next_cursor`.
10. **Field language**: All backlog fields are in English.
