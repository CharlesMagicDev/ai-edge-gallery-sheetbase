# SheetBase RAG — an Agent Skill for Google AI Edge Gallery

Keyword retrieval over any **public Google Sheet**, executed inside the on-device WebView.
One `SKILL.md`, one `scripts/index.html`, Vanilla JavaScript. No API key, no OAuth, no backend,
no build step — just a URL to import.

## Import on your phone

1. Install **Google AI Edge Gallery** (Android / iOS 17+) and download **Gemma 4 E2B** (fast) or
   **E4B** (better reasoning).
2. Open **Agent Skills** → **Try it** with the model loaded.
3. Tap the **Skills** icon → **+** → **Load skill from URL**.
4. Paste this folder URL (no `SKILL.md` suffix, GitHub Pages serves the MIME types the WebView needs):

   ```
   https://charlesmagicdev.github.io/ai-edge-gallery-sheetbase/skills/sheetbase-rag
   ```

5. Add, then just ask: *"Search the sheet https://docs.google.com/spreadsheets/d/…/edit for Montpellier."*

The sheet must be shared as **Anyone with the link — Viewer**.

## What it does

`run_js` calls `window['ai_edge_gallery_get_result'](data)` with a JSON string such as:

```json
{"sheetUrl":"https://docs.google.com/spreadsheets/d/<id>/edit#gid=0","keyword":"montpellier"}
```

It streams the tab as CSV (gviz, then `export?format=csv`, then a published link), parses it with an
RFC 4180 state machine (quoted commas, doubled quotes, embedded newlines, BOM), and returns a compact
text summary plus structured rows: every match carries its **row number**, the **cell reference**
(`D14`) and the **column name** of each matching cell, so the model can tell the user exactly where
to look and edit.

Optional parameters: `limit`, `maxLength`, `columns` (by header or letter), `gid`, `timeoutMs`.
Nothing is capped by default.

## Limits, stated plainly

- Retrieval is **lexical** (accent- and case-insensitive substring matching), not embedding-based:
  no vector index, no model call.
- The rows live in the Google Sheet, not in the phone's storage: the WebView needs a network fetch at
  call time. What is on-device is the logic, the parsing and the reasoning.
- The tool's human-readable strings are in French (the author's locale); values are returned verbatim.

## Layout

```
.nojekyll                              # GitHub Pages: serve files as-is
skills/
└── sheetbase-rag/
    ├── SKILL.md                       # YAML frontmatter (name, description) + instructions
    └── scripts/
        └── index.html                 # window['ai_edge_gallery_get_result'] = async (data, secret)
```

## Tested

Headless Chromium, calling the function exactly the way the app does:

- nominal keyword, accented match (`Dùpont` found by `dupont`), empty keyword
- missing `sheetUrl` and raw-URL input
- private sheet (403) → actionable French error
- a real public sheet: 30 rows, range `A1:F31`, 1 match on `alexandra`, cell reference returned
- CSV edge cases: `"Dupont, Jean"` and `"Client ""fidèle"", depuis 2019"`
