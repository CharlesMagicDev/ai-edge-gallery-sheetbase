---
name: sheetbase-rag
description: Search rows in a public Google Sheet by URL and keyword. Use whenever the message contains a docs.google.com/spreadsheets link and asks to find, filter, list or look up data in it.
---

# SheetBase RAG

**This skill applies whenever the message contains a Google Sheets link** (`docs.google.com/spreadsheets/d/…`).
In that case the skill IS the relevant one: do not answer "No relevant skills found". Call `run_js` now.

## The call

```
run_js(
  skillName  = "sheetbase-rag",
  scriptName = "index.html",
  data       = "{\"sheetUrl\":\"<full url>\",\"keyword\":\"<term>\"}"
)
```

`data` is a **JSON string**, not an object. `keyword` may be empty (it then lists the first 10 rows so
you can see the columns).

Optional keys inside `data`: `limit` (max rows), `maxLength` (truncate each cell), `columns` (e.g.
`["Nom","C"]`), `gid` (tab), `timeoutMs`. Nothing is capped unless you ask.

## Reading the answer

- Success → `result`: one summary line, then one line per match, each with its **row number** and,
  for every matching cell, the **cell reference** (`B17`) and column name. Quote values exactly and
  give the user the cell reference.
- Row 1 is the header row, so `row 17` is line 17 in Google Sheets.
- `count` = number of matching rows. Never invent one: if `count` is 0, say so and suggest a shorter
  keyword (search ignores accents and case).
- `error` comes back in French: relay it in the user's language. If it mentions sharing, tell the
  user to set the sheet to « Anyone with the link — Viewer » and to paste the complete link.
- Answer in the user's language.
