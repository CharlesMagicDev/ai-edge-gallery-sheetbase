# Agent Skills for Google AI Edge Gallery

Two on-device skills for [Google AI Edge Gallery](https://github.com/google-ai-edge/gallery), both
plain Vanilla JavaScript running inside the WebView. No API key, no OAuth, no backend, no build step —
just a URL to import.

- **`sheetbase-rag`** — keyword retrieval over any **public Google Sheet** (network at call time).
- **`localbase`** — a small **on-device database** the agent creates, fills, searches and edits;
  data persists in the WebView storage, nothing leaves the phone.

## Import on your phone

1. Install **Google AI Edge Gallery** (Android / iOS 17+) and download **Gemma 4 E2B** (fast) or
   **E4B** (better reasoning).
2. Open **Agent Skills** → **Try it** with the model loaded.
3. Tap the **Skills** icon → **+** → **Load skill from URL**.
4. Paste the folder URL of the skill you want (no `SKILL.md` suffix, GitHub Pages serves the MIME
   types the WebView needs):

   ```
   https://charlesmagicdev.github.io/ai-edge-gallery-sheetbase/skills/sheetbase-rag
   https://charlesmagicdev.github.io/ai-edge-gallery-sheetbase/skills/localbase
   ```

5. Just ask, e.g. *"Search the sheet https://docs.google.com/spreadsheets/d/…/edit for Montpellier."*

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
Nothing is capped by default — with one deliberate exception: an empty keyword with no `limit` is a
discovery call, so it lists the first 10 rows and says so, instead of flooding a small context window
with the whole sheet.

A sheet whose cells were pasted as raw CSV into a single column is detected and re-split into real
columns, so results come back as `B17 « Nom »` instead of one giant `A6`.

## localbase — the on-device database

Same contract, different job: the agent keeps structured data **on the phone**.

```json
{"action":"create","table":"clients","columns":["Nom","Ville","Tel"]}
{"action":"insert","table":"clients","rows":[{"Nom":"Dupont","Ville":"Montpellier"}]}
{"action":"query","table":"clients","keyword":"montpellier"}
{"action":"update","table":"clients","id":2,"values":{"Tel":"0600000000"}}
{"action":"delete","table":"clients","id":3}
```

Actions: `schema`, `create`, `insert`, `query`, `update`, `delete`, `drop`, `export`, `import`,
`reset`. Every row gets a stable `_id` used by `update`/`delete`; destructive actions require
`confirm:true`. Storage is one `localStorage` key (`localbase.v1`) holding JSON, rewritten on each
write, guarded at 4 MB — the WebView origin is shared by every skill and capped near 5 MB, hence the
prefix and the `export`/`drop` path.

`domStorageEnabled = true` in `GalleryWebView.kt` is what makes this possible; the origin is
`https://appassets.androidplatform.net`, stable across calls and app restarts.

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
├── sheetbase-rag/
│   ├── SKILL.md                       # YAML frontmatter (name, description) + instructions
│   └── scripts/
│       └── index.html                 # window['ai_edge_gallery_get_result'] = async (data, secret)
└── localbase/
    ├── SKILL.md
    └── scripts/
        └── index.html                 # same contract, localStorage instead of a Sheet
```

## Tested

Headless Chromium, calling the function exactly the way the app does:

- nominal keyword, accented match (`Dùpont` found by `dupont`), empty keyword
- missing `sheetUrl` and raw-URL input
- private sheet (403) → actionable French error
- a real public sheet: 30 rows, range `A1:F31`, 1 match on `alexandra`, cell reference returned
- CSV edge cases: `"Dupont, Jean"` and `"Client ""fidèle"", depuis 2019"`
- localbase, CRUD end to end, then a **browser restart** on the same profile: table, rows, edit and
  delete all survived; `export` → `reset` → `import` restored the exact state; unknown column,
  unknown id, missing `confirm` and non-JSON input all return actionable errors, no JS exception
