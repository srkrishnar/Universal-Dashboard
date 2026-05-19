# CLAUDE CODE — ONE-SHOT BUILD PROMPT
# Paste this entire prompt into Claude Code after `claude` starts in your project folder.
# Pre-requisite: CLAUDE.md is already in the project root.
# ─────────────────────────────────────────────────────────────────────────────

Read CLAUDE.md fully before writing any code.

Build `dashboard.html` — a complete, single-file, AI-powered universal dashboard application.

## What to build

A self-contained HTML file that:

1. On first load, shows an **API Key Modal** — full-screen overlay, input field for
   Anthropic API key, "Save & Continue" button. Store key in sessionStorage only.
   Never hardcode or log the key. Show masked once saved with a "Change" link.

2. After key entry, shows an **Upload Landing State** — centred card with a dashed
   drop-zone, drag-and-drop support, click-to-browse, accepts `.csv` and `.xlsx` only.
   Tagline: "Upload any dataset. Claude builds your dashboard instantly."

3. On file upload:
   a. Parse with PapaParse (CSV) or SheetJS (XLSX)
   b. Run `classifyColumns()` — classify each column as numeric/date/categorical/id
   c. Show loading spinner with message "Claude is analysing your data…"
   d. Call Claude API (`claude-sonnet-4-6`) with `buildApiPrompt()` — pass filename,
      column classification summary, first 20 rows as JSON
   e. Receive DashboardSpec JSON back from Claude
   f. Run `validateSpec()` — check all required keys present, all column refs exist in data
   g. Set `fmt = buildFormatter(spec)` — domain-aware number formatter
   h. Render full dashboard: KPIs, charts, insights, table, filters
   i. Show export buttons (Download HTML, Download PDF)

4. **Dashboard renders spec-driven** — every KPI, chart, filter, insight driven by
   DashboardSpec JSON. Zero hardcoded column names. Works for any dataset domain:
   sales, HR, QA metrics, finance, logistics, healthcare, or any other.

5. **Filters bar** — one dropdown per entry in `spec.filters`. On change, filters
   `RAW_DATA` and calls `renderAll(spec, FILTERED_DATA)`. Reset button restores full data.

6. **Export** — "Download HTML" saves `document.documentElement.outerHTML` as `.html`.
   "Download PDF" uses html2canvas + jsPDF landscape snapshot.

7. **Re-import** — "Import Dataset" button in header accepts new file, destroys all
   existing charts, reruns full pipeline from step 3.

## Exact libraries to load (CDN, these versions only)

```html
<link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@600;700&family=DM+Sans:wght@300;400;500;600&display=swap" rel="stylesheet"/>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chartjs-plugin-datalabels@2.2.0/dist/chartjs-plugin-datalabels.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/jspdf@2.5.1/dist/jspdf.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
```

## Design system (from CLAUDE.md — reproduce exactly)

- Background `#FAF7F2`, surface `#FFFFFF`, surface2 `#F5F0E8`
- Palette: brown `#6B4C35`, caramel `#C48A3F`, olive `#7A8C5C`,
  orange `#E87040`, purple-deep `#9B88B0`
- All values as CSS variables in `:root` — never inline hex
- Fonts: Playfair Display for KPI numbers + card titles; DM Sans for everything else
- Cards: `border-radius: 16px`, `box-shadow: 0 4px 24px rgba(107,76,53,0.08)`
- KPI cards: 3px accent top-border, hover `translateY(-2px)`, staggered fade-up on load
- Header: sticky, `Import Dataset` button caramel-to-orange gradient top right
- Filters bar: sticky below header, dropdowns with custom arrow, Reset button right-aligned

## Chart requirements (all from CLAUDE.md)

- Register ChartDataLabels globally; set display:false globally; enable per chart
- Every chart has datalabels:
  - Line: pill label above each point (fmt value, white bg, caramel border)
  - Vertical bar: pill label on top of bar
  - Horizontal bar: adaptive — white inside wide bars, dark + pill outside narrow bars
  - Donut: % share inside each slice, white bold, text-shadow
- Tooltip styled: white bg, brown title, muted body, border `#EDE5D8`, radius 10
- Chart color cycle: `#C48A3F, #7A8C5C, #E87040, #6B4C35, #9B88B0, #8B6650, #9AAD74, #E0A84F`
- `maintainAspectRatio: false` on all charts
- `destroyChart(id)` before every rebuild

## Chart layout grid

Row 1: `grid-template-columns: 2fr 1fr`   → line chart (wide) + donut (narrow)
Row 2: `grid-template-columns: 1fr 1fr 1fr` → 3 equal charts
Row 3: `grid-template-columns: 1fr 1fr`   → 2 equal charts (if spec has 7 charts)
All rows: `gap: 20px; margin-bottom: 28px`

## KPI cards

- 6 cards in `grid-template-columns: repeat(auto-fit, minmax(200px, 1fr))`
- Each card renders from `spec.kpis[i]`: label, computed value, sub-text, badge
- Aggregation logic: "count" → row count, "sum" → column total,
  "average" → column mean, "ratio" → (count matching value / total) × 100

## Insights section

- Render `spec.insights` array as a 2-col grid of bullet cards
- Each card: left accent border (cycles palette), emoji, text with `<strong>` tags
- 8 minimum insights — already generated by Claude API in the spec

## Top 10 Table

- Columns from `spec.tableColumns`
- Sorted descending by `spec.tableValueColumn`
- Status column (spec.tableStatusColumn) renders as pill:
  green pill if value in `spec.tableStatusPositive`,
  orange pill if value in `spec.tableStatusNegative`,
  grey otherwise

## Error and loading states

- Loading: full-area overlay with spinner + "Claude is analysing your data…"
- Toast component: bottom-right, auto-dismiss 4s, used for all errors:
  "Claude API error — check your API key and try again"
  "Could not parse AI response — try re-uploading the file"
  "No data rows found in the uploaded file"
  "Invalid file type — please upload a .csv or .xlsx file"

## Code quality rules (from CLAUDE.md)

- Clean, commented, never minified
- Section header comments: `/* ═══════ SECTION NAME ══════════ */`
- JSDoc on every function
- No `console.log` — use `function debug(msg) { if(DEBUG_MODE) console.log(msg); }`
- CSS variables only — no hex outside `:root`
- No `innerHTML` for user data — use `textContent` or sanitise
- `DEBUG_MODE = false` in production code at top of script

## Exact function signatures to implement

```javascript
classifyColumns(rows)                    → { numeric[], date[], categorical[], id[] }
buildApiPrompt(filename, colSummary, rows) → string
callClaudeAPI(prompt)                    → Promise<DashboardSpec>
validateSpec(spec, columnNames)          → void (throws on invalid)
buildFormatter(spec)                     → function(n) → string
renderAll(spec, data)                    → void
renderKPIs(spec, data)                   → void
renderCharts(spec, data)                 → void  // iterates spec.charts array
renderInsights(spec)                     → void
renderTable(spec, data)                  → void
populateFilters(spec, data)              → void
applyFilters()                           → void  // reads dropdowns, calls renderAll
resetFilters()                           → void
destroyChart(id)                         → void
parseFile(file)                          → Promise<{ rows, filename }>
exportHTML()                             → void
exportPDF()                              → Promise<void>
getApiKey()                              → string | null
setApiKey(key)                           → void
showLoading(msg)                         → void
hideLoading()                            → void
showToast(msg, type)                     → void  // type: 'error' | 'success' | 'info'
debug(msg)                               → void
```

## Reference: the existing sales dashboard

A working reference implementation already exists at `test-data/sales.xlsx` with the
completed sales dashboard as visual reference. Use that dashboard's:
- Visual layout, spacing, card proportions
- Colour application and hover effects
- Animation timing (staggered 50ms per card)
- Table pill badge style
- Header layout and button style

The new `dashboard.html` must match or exceed that visual quality — but driven
entirely by DashboardSpec JSON, not hardcoded sales data.

## Deliverable

One file: `dashboard.html`
All JS inline. All CSS inline. Opens by double-click. No server. No build.
Test by uploading: sales Excel → should produce sales dashboard.
Test by uploading: any HR CSV → should produce HR dashboard.
Test by uploading: any QA metrics CSV → should produce QA/dev dashboard.
