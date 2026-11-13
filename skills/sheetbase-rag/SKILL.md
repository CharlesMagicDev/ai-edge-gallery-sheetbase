---
name: sheetbase-rag
description: Search and retrieve rows from a public Google Sheet by URL and keyword. Use when the user asks to find, filter or look up data in a spreadsheet.
---

# SheetBase RAG

Keyword retrieval over a public Google Sheet, executed entirely inside the hidden WebView of
Google AI Edge Gallery. One `SKILL.md` and one `scripts/index.html`, Vanilla JavaScript, no build
step, no API key, no server, no OAuth.

## Instructions

When the user asks to search, filter or look something up in a Google Sheet, call the `run_js` tool
with the script `index.html` and a **JSON string** for `data` containing:

- **sheetUrl**: the full public URL of the Google Sheet
  (`https://docs.google.com/spreadsheets/d/<id>/edit#gid=0`). A published link
  (`/spreadsheets/d/e/2PACX…/pubhtml`) and a bare sheet id also work.
- **keyword**: the search term. It may be empty, to list the first rows and discover the columns.

```json
{"sheetUrl":"https://docs.google.com/spreadsheets/d/<id>/edit#gid=0","keyword":"montpellier"}
```

The tool returns a JSON string containing **either**:

- `result` — a readable summary, then one line per match. Each match starts with its **row number**
  and, for every cell containing the keyword, the **cell reference** and column name:
  `3. ligne 14 · D14 « Home State » = CA | Student Name = John · …`. The summary also gives the
  sheet range (`A1:F31`) and the column map (`A = Student Name | B = Gender | …`). `kind`, `count`,
  `columns` and `rows` are included, each row carrying `row`, `cells` (`["D14"]`), `columns`
  (`["Home State"]`) and `data`.
- `error` — a French message explaining what failed.

Matching is lexical: case- and accent-insensitive substring search across every column, no
embeddings and no index. The sheet is streamed as CSV at call time, so the sheet must be shared as
« Anyone with the link — Viewer ».

## Optional parameters

All optional: without them the search is exhaustive (every matching row, every cell in full, all
columns). One exception: an **empty keyword with no `limit` is a discovery call** — the first 10 rows
are listed and the summary says so ("découverte : les 10 premières sont listées sur 82"), so a small
model is not drowned before it can ask for a keyword.

- **limit**: return at most N matching rows (`"limit": 20`).
- **maxLength**: truncate each cell to N characters (`"maxLength": 120`).
- **columns**: search only in these columns, by header name or letter (`"columns": ["City","Name"]`
  or `["C","A"]`).
- **gid**: read a specific tab (`"gid": "123456789"`), otherwise the `#gid=` of the URL, otherwise
  the first tab.
- **timeoutMs**: download timeout per attempt (default 20000).

## Answering

- Quote the values **exactly** as returned. When the user must check or edit the sheet, give the
  **cell reference** (`D14`) or at least the row number (`row 14`) and the column name — the agent
  has both, so use them instead of describing "around the middle of the sheet".
- Row numbers count the sheet as displayed: row 1 is the header row, so `row 14` is the 14th row in
  Google Sheets, not the 14th data row.
- Never invent a row: if `count` is 0, say so and suggest a shorter keyword (search is accent- and
  case-insensitive and covers every column).
- Error messages come back in French: relay them in the user's language. If an `error` mentions
  sharing, tell the user to set the sheet to « Anyone with the link — Viewer » and to paste the
  complete link.
- Nothing is trimmed by default: every matching row is returned, complete. Only say "first N" when
  you asked for a limit yourself, and give the total from `count`.
