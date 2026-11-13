---
name: localbase
description: Local on-device table database the agent can create, fill, search and edit. Use when the user wants to record, list, track or look up structured data that must persist between messages.
---

# LocalBase

A tiny table database that lives **on the phone**, inside the skill's WebView storage. No account, no
server, no API key. The agent creates tables on demand, adds rows, searches them by keyword, edits and
deletes them, and the data is still there in the next conversation.

**This skill applies whenever the user wants to keep structured data** — a list, a log, a tracker, a
small inventory, a contact book. In that case the skill IS the relevant one: do not answer
"No relevant skills found". Call `run_js` now.

## The call

```
run_js(
  skillName  = "localbase",
  scriptName = "index.html",
  data       = "{\"action\":\"insert\",\"table\":\"clients\",\"rows\":[{\"Nom\":\"Dupont\",\"Ville\":\"Montpellier\"}]}"
)
```

`data` is a **JSON string**, not an object. **One action per call.**

## After every call

- Every answer carries a `next` field: **one sentence telling you what to do or say next**. Follow it.
- **Never answer "No relevant skills found" after a call.** That sentence belongs to the app's skill
  selection step, not to a result. If you received a `result` or an `error`, answer the user in their
  language, from that content.
- An empty database is **not an error**: `query` on a table that does not exist yet returns
  `empty: true` with a `result` explaining it. Say so and propose to create the table — do not stop.

## Actions

| action | fields | what it does |
|---|---|---|
| `schema` | — | lists tables, row counts, columns. **Start here when you don't know the state.** |
| `create` | `table`, `columns:["Nom","Ville"]` | creates a table (`ifNotExists:true` to tolerate an existing one) |
| `insert` | `table`, `rows:[{…}]` | appends rows, returns the new `ids` |
| `query` | `table`, `keyword`, optional `columns`, `limit` | keyword search, accent- and case-insensitive |
| `update` | `table`, `id`, `values:{…}` | edits one row by its `_id` |
| `delete` | `table`, `id` | removes one row by its `_id` |
| `drop` | `table`, `confirm:true` | deletes a whole table |
| `export` | — | returns the whole database as JSON (backup) |
| `import` | `json`, `confirm:true` | restores a backup, replacing everything |
| `reset` | `confirm:true` | wipes everything |

## Working rules

- **Never invent an `_id`.** Read it from the last `query`/`insert` answer, or call `query` first.
- **Check before creating**: `schema` costs one call and avoids "table already exists".
- An empty `keyword` lists the first 10 rows — use it to see what a table contains.
- Destructive actions (`drop`, `import`, `reset`) need `confirm:true`; ask the user before sending it.
- After an `insert`, confirm the count and the ids you got back. Never claim a row was saved without
  a successful answer.

## Reading the answer

- Success → `result`: a short summary, then one line per row: `2. #7 · Nom = Dupont | Ville = Montpellier`.
  `#7` is the row id for `update`/`delete`. `count` is the number of matching rows.
- `error` comes back in French: relay it in the user's language. It always names the fix (missing
  column, unknown table, list of available columns, quota).
- Answer in the user's language, quoting values exactly as returned.

## Limits

- The storage quota is about 5 MB for the whole WebView origin, shared with the other skills.
  `export` + `drop` is the way out when it fills up.
- Data lives on this device only. Uninstalling the app or clearing its data erases it: tell the user
  to keep an `export` if the data matters.
