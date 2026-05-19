# CLAUDE.md — Universal AI-Powered Dashboard Engine
# Auto-loaded by Claude Code at every session start.
# This is the single source of truth. Never guess. Never deviate.
# ─────────────────────────────────────────────────────────────────────────────

## Project Identity

- **Name**: Universal Dashboard Engine
- **Type**: Single-file HTML application — zero build step, zero backend
- **Purpose**: Upload any CSV/XLSX → Claude API analyses columns → renders
  a domain-aware, fully interactive dashboard automatically
- **Output**: One self-contained `dashboard.html` file the user can download/share
- **Primary Tech**: Vanilla HTML + CSS + JavaScript (ES2020)
- **AI Layer**: Anthropic Claude API (`claude-sonnet-4-6`) via `fetch` in browser

---

## Repository Layout

```
universal-dashboard/
├── CLAUDE.md                  ← this file (Claude Code reads first)
├── dashboard.html             ← THE deliverable — entire app lives here
├── docs/
│   ├── architecture.md        ← component map, data flow, API contract
│   ├── column-classifier.md   ← rules for type detection + domain guessing
│   ├── chart-standards.md     ← Chart.js config standards per chart type
│   └── design-system.md       ← CSS variables, palette, spacing, typography
└── test-data/
    ├── sales.xlsx             ← reference dataset (already built)
    ├── hr-recruitment.csv     ← HR test dataset
    └── qa-metrics.csv         ← QA/sprint metrics test dataset
```

---

## Non-Negotiable Architecture Rules

### Single File Law
- `dashboard.html` contains ALL CSS, ALL JS, ALL markup — inline, no imports
- Zero npm, zero webpack, zero build step
- CDN scripts only (Chart.js, ChartDataLabels, PapaParse, SheetJS, jsPDF, html2canvas)
- Must open in any browser by double-click — no localhost required

### Two-Phase Rendering
```
Phase 1 — FILE PARSE (browser, instant)
  User uploads CSV/XLSX
  → PapaParse / SheetJS extracts all rows
  → Column Classifier runs (JS, no API)
  → First 20 rows + column summary sent to Claude API

Phase 2 — AI SPEC (Claude API, ~2 seconds)
  Claude returns DashboardSpec JSON
  → Renderer reads spec
  → Builds KPI cards, charts, filters, insights, table
  → Everything driven by spec — zero hardcoded column names
```

### DashboardSpec JSON Contract
Claude API MUST return exactly this shape. Validate before rendering.
```json
{
  "domain": "HR Recruitment",
  "domainIcon": "👥",
  "title": "HR Recruitment Dashboard",
  "subtitle": "Recruitment pipeline analysis",
  "currencySymbol": "",
  "valueFormat": "number",
  "kpis": [
    {
      "label": "Total Openings",
      "column": "Job ID",
      "aggregation": "count",
      "icon": "📋",
      "accentClass": "accent-caramel",
      "badgeText": "All Roles",
      "badgeClass": "neutral"
    }
  ],
  "charts": [
    {
      "id": "chart1",
      "type": "bar",
      "title": "Openings by Department",
      "description": "Headcount demand per department",
      "groupBy": "Department",
      "valueColumn": "Job ID",
      "aggregation": "count",
      "tag": "Bar"
    },
    {
      "id": "chart2",
      "type": "donut",
      "title": "Status Breakdown",
      "description": "Open vs Closed positions",
      "groupBy": "Status",
      "valueColumn": "Job ID",
      "aggregation": "count",
      "tag": "Donut"
    },
    {
      "id": "chart3",
      "type": "line",
      "title": "Monthly Openings Trend",
      "description": "Hiring volume over time",
      "groupBy": "Month",
      "valueColumn": "Job ID",
      "aggregation": "count",
      "tag": "Line"
    },
    {
      "id": "chart4",
      "type": "horizontalBar",
      "title": "Top Roles by Volume",
      "description": "Most frequently opened positions",
      "groupBy": "Role",
      "valueColumn": "Job ID",
      "aggregation": "count",
      "tag": "Horizontal Bar"
    }
  ],
  "filters": [
    { "column": "Department", "label": "Department" },
    { "column": "Status", "label": "Status" },
    { "column": "Recruiter", "label": "Recruiter" }
  ],
  "insights": [
    {
      "emoji": "📈",
      "text": "<strong>Engineering</strong> has the highest open position count at <strong>42</strong> — 31% of all openings."
    }
  ],
  "tableColumns": ["Job ID", "Role", "Department", "Status", "Recruiter", "Open Date", "Days to Fill"],
  "tableValueColumn": "Days to Fill",
  "tableStatusColumn": "Status",
  "tableStatusPositive": ["Closed", "Filled"],
  "tableStatusNegative": ["Open", "Pending"]
}
```

---

## CDN Scripts — exact versions, do not change

```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chartjs-plugin-datalabels@2.2.0/dist/chartjs-plugin-datalabels.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/jspdf@2.5.1/dist/jspdf.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
```

---

## Design System — CSS Variables (never override inline)

```css
:root {
  --bg:           #FAF7F2;
  --surface:      #FFFFFF;
  --surface2:     #F5F0E8;
  --brown:        #6B4C35;
  --brown-light:  #8B6650;
  --caramel:      #C48A3F;
  --caramel-light:#E0A84F;
  --olive:        #7A8C5C;
  --olive-light:  #9AAD74;
  --orange:       #E87040;
  --orange-light: #F08C5A;
  --purple:       #C9B8D8;
  --purple-deep:  #9B88B0;
  --text-primary: #2C1F14;
  --text-secondary:#7A6555;
  --text-muted:   #B09880;
  --border:       #EDE5D8;
  --shadow:       0 4px 24px rgba(107,76,53,0.08);
  --shadow-lg:    0 8px 40px rgba(107,76,53,0.13);
  --radius:       16px;
  --radius-sm:    10px;
}
```

**Fonts**: Playfair Display (700) for KPI values + titles; DM Sans (400/500/600) for body.
Load from Google Fonts. Never use Inter, Roboto, Arial, or system fonts.

**Chart color cycle** (always in this order):
`#C48A3F, #7A8C5C, #E87040, #6B4C35, #9B88B0, #8B6650, #9AAD74, #E0A84F`

---

## Chart.js Standards — apply to every chart, no exceptions

### Bootstrap (runs once at init)
```javascript
Chart.register(ChartDataLabels);
Chart.defaults.set('plugins.datalabels', { display: false }); // off globally
Chart.defaults.font.family = "'DM Sans', sans-serif";
Chart.defaults.color = '#7A6555';
Chart.defaults.plugins.tooltip.backgroundColor = '#FFFFFF';
Chart.defaults.plugins.tooltip.titleColor = '#2C1F14';
Chart.defaults.plugins.tooltip.bodyColor = '#7A6555';
Chart.defaults.plugins.tooltip.borderColor = '#EDE5D8';
Chart.defaults.plugins.tooltip.borderWidth = 1;
Chart.defaults.plugins.tooltip.padding = 10;
Chart.defaults.plugins.tooltip.cornerRadius = 10;
```

### Data Label Configs per Chart Type

**Line** — pill label above each point:
```javascript
datalabels: {
  display: true,
  formatter: v => fmt(v),
  color: '#6B4C35', font: { size: 10, weight: '600' },
  anchor: 'end', align: 'top', offset: 4,
  backgroundColor: 'rgba(255,255,255,0.85)',
  borderRadius: 5, borderWidth: 1, borderColor: 'rgba(196,138,63,0.3)',
  clamp: true,
}
// layout: { padding: { top: 28 } }
```

**Vertical Bar** — pill label on top:
```javascript
datalabels: {
  display: true,
  formatter: v => fmt(v),
  color: '#6B4C35', font: { size: 11, weight: '600' },
  anchor: 'end', align: 'top', offset: 4,
  backgroundColor: 'rgba(255,255,255,0.88)',
  borderRadius: 5, borderWidth: 1, borderColor: 'rgba(107,76,53,0.15)',
}
// layout: { padding: { top: 30 } }
```

**Horizontal Bar** — adaptive label (white inside wide bars, dark outside narrow):
```javascript
datalabels: {
  display: true, formatter: v => fmt(v),
  color: ctx => ctx.dataset.data[ctx.dataIndex] / Math.max(...ctx.dataset.data) > 0.45
    ? '#FFFFFF' : '#6B4C35',
  font: { size: 11, weight: '700' },
  anchor: 'end',
  align: ctx => ctx.dataset.data[ctx.dataIndex] / Math.max(...ctx.dataset.data) > 0.45
    ? 'start' : 'end',
  offset: 6,
  backgroundColor: ctx => ctx.dataset.data[ctx.dataIndex] / Math.max(...ctx.dataset.data) > 0.45
    ? 'transparent' : 'rgba(255,255,255,0.88)',
  borderRadius: 4, borderWidth: 1, borderColor: 'rgba(107,76,53,0.15)',
}
// layout: { padding: { right: 80 } }
```

**Donut** — % inside each slice:
```javascript
datalabels: {
  display: true,
  formatter: (v, ctx) => {
    const t = ctx.dataset.data.reduce((a,b)=>a+b,0);
    return ((v/t)*100).toFixed(1)+'%';
  },
  color: '#FFFFFF', font: { size: 12, weight: '700' },
  textShadowColor: 'rgba(0,0,0,0.25)', textShadowBlur: 4,
}
```

---

## Number Formatting — `fmt(v)` function

The `fmt` function is domain-aware. Set after Claude API returns the spec:
```javascript
function buildFormatter(spec) {
  if (spec.valueFormat === 'currency') {
    return n => {
      const sym = spec.currencySymbol || '₹';
      if (n >= 10000000) return sym + (n/10000000).toFixed(2) + ' Cr';
      if (n >= 100000)   return sym + (n/100000).toFixed(1) + ' L';
      if (n >= 1000)     return sym + (n/1000).toFixed(1) + 'K';
      return sym + n.toLocaleString();
    };
  }
  if (spec.valueFormat === 'days')       return n => n.toFixed(1) + 'd';
  if (spec.valueFormat === 'percentage') return n => n.toFixed(1) + '%';
  return n => {
    if (n >= 1000000) return (n/1000000).toFixed(1) + 'M';
    if (n >= 1000)    return (n/1000).toFixed(1) + 'K';
    return Math.round(n).toLocaleString();
  };
}
let fmt = n => n; // replaced after spec loads
```

---

## Column Classifier (runs in browser before API call)

```javascript
function classifyColumns(rows) {
  const cols = Object.keys(rows[0]);
  const result = { numeric: [], date: [], categorical: [], id: [] };

  cols.forEach(col => {
    const sample = rows.slice(0,50).map(r=>r[col]).filter(v=>v!==''&&v!=null);
    const isId = /id|code|ref|number|no\.?$/i.test(col) && new Set(sample).size===sample.length;
    if (isId) { result.id.push(col); return; }
    const numericCount = sample.filter(v=>!isNaN(parseFloat(v))).length;
    if (numericCount/sample.length > 0.85) { result.numeric.push(col); return; }
    const dateCount = sample.filter(v=>!isNaN(Date.parse(v))).length;
    if (dateCount/sample.length > 0.85) { result.date.push(col); return; }
    const uniqueRatio = new Set(sample).size/sample.length;
    if (uniqueRatio < 0.4 || new Set(sample).size <= 25) result.categorical.push(col);
  });
  return result;
}
```

---

## Claude API Prompt — sent after column classification

```javascript
function buildApiPrompt(filename, colSummary, sampleRows) {
  return `You are a data analyst. Analyse this dataset and return ONLY a valid JSON object matching the DashboardSpec schema. No markdown, no explanation, no backticks — raw JSON only.

DATASET: "${filename}"
COLUMNS: ${JSON.stringify(colSummary)}
SAMPLE (first 20 rows): ${JSON.stringify(sampleRows.slice(0,20))}

RULES:
1. Detect the business domain (Sales, HR, QA, Finance, Logistics, Healthcare, etc.)
2. Select 4-6 most meaningful KPIs — prefer count, sum, average, ratio depending on column type
3. Select 4 charts — always include: one time/trend chart, one categorical bar, one distribution donut, one ranking
4. Select 3-4 filter columns — must be categorical with < 25 unique values
5. Write 8 specific data-driven insights referencing actual column names and values from the sample
6. Set valueFormat to: "currency" if monetary, "days" if duration, "percentage" if ratios, "number" otherwise
7. tableStatusPositive and tableStatusNegative must reflect actual values in the status column
8. All column references must exactly match column names in the dataset

Return the DashboardSpec JSON now.`;
}
```

---

## API Call Handler

```javascript
async function callClaudeAPI(prompt) {
  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': getApiKey(),
      'anthropic-version': '2023-06-01',
      'anthropic-dangerous-direct-browser-calls': 'true'
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-6',
      max_tokens: 2000,
      messages: [{ role: 'user', content: prompt }]
    })
  });
  const data = await response.json();
  const raw = data.content[0].text.trim();
  const clean = raw.replace(/^```json\n?/,'').replace(/\n?```$/,'').trim();
  return JSON.parse(clean);
}
```

---

## API Key Handling — Security Rule

- **Never hardcode the API key in source**
- On first load: show a modal asking user to enter their Anthropic API key
- Store in `sessionStorage` only — cleared when tab closes
- Display masked (•••••) once entered; "Change Key" button to reset
- Show clear warning: "Your key is stored in this browser tab only"

```javascript
function getApiKey() { return sessionStorage.getItem('ANTHROPIC_KEY') || null; }
function setApiKey(key) { sessionStorage.setItem('ANTHROPIC_KEY', key.trim()); }
```

---

## Export Features

### HTML Export
```javascript
function exportHTML() {
  const html = '<!DOCTYPE html>' + document.documentElement.outerHTML;
  const blob = new Blob([html], { type: 'text/html' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = CURRENT_TITLE.replace(/\s+/g,'_') + '_dashboard.html';
  a.click();
}
```

### PDF Export
```javascript
async function exportPDF() {
  const canvas = await html2canvas(document.getElementById('dashboardMain'),
    { scale: 2, backgroundColor: '#FAF7F2', useCORS: true });
  const { jsPDF } = window.jspdf;
  const pdf = new jsPDF({ orientation: 'landscape', unit: 'px',
    format: [canvas.width/2, canvas.height/2] });
  pdf.addImage(canvas.toDataURL('image/png'), 'PNG', 0, 0, canvas.width/2, canvas.height/2);
  pdf.save(CURRENT_TITLE.replace(/\s+/g,'_') + '_dashboard.pdf');
}
```

---

## Complete JS Execution Flow

```
1.  Page loads → show API key modal if no key in sessionStorage
2.  User enters key → modal closes → show upload landing state
3.  User uploads file → showLoading()
4.  parseFile(file) → RAW_DATA array + filename
5.  classifyColumns(RAW_DATA) → colSummary object
6.  buildApiPrompt(filename, colSummary, RAW_DATA) → prompt string
7.  callClaudeAPI(prompt) → DashboardSpec JSON
8.  validateSpec(spec) → throw if malformed
9.  fmt = buildFormatter(spec) → set global formatter
10. populateFilters(spec, RAW_DATA) → build filter dropdowns
11. renderAll(spec, RAW_DATA) →
        renderKPIs(spec, data)
        renderCharts(spec, data)   ← iterates spec.charts array
        renderInsights(spec)
        renderTable(spec, data)
12. hideLoading() → show dashboard
13. On filter change → applyFilters() → renderAll(spec, FILTERED_DATA)
14. On new import → go to step 3, destroy all charts first
```

---

## Error Handling Rules

- API key missing → show key modal, do not proceed
- API call fails → toast: "Claude API error — check your API key and try again"
- JSON parse fails → toast: "Could not parse AI response — try re-uploading"
- Empty dataset → toast: "No data rows found in file"
- Column mismatch → `validateSpec()` remaps to nearest column name match
- Never show raw JS error objects — always human-readable toast messages

---

## UI Components Required

| Component        | Description |
|------------------|-------------|
| API Key Modal    | Full-screen overlay; input + save; masked display after entry |
| Upload Landing   | Centred drop-zone; dashed border; drag-and-drop + click |
| Loading State    | Spinner + "Claude is analysing your data…"; covers dashboard area |
| Header           | Logo + title + subtitle + Import button + Export (HTML + PDF) buttons |
| Filters Bar      | Sticky; dropdowns per filter column + Reset button |
| KPI Grid         | 3-col desktop, 2-col tablet, 1-col mobile; 4–6 cards |
| Charts Grid      | Row 1: line (2fr) + donut (1fr); Row 2: 3 equal; Row 3: 2 equal |
| Insights Section | 2-col grid of bullet cards; left border cycles palette |
| Top 10 Table     | Scrollable; pill badges; hit/miss status column |
| Toast            | Bottom-right; auto-dismiss 4 seconds |

---

## Generation Rules for Claude Code

1. Read this CLAUDE.md fully before writing a single line.
2. The DashboardSpec JSON contract is immutable — never change its shape.
3. `renderCharts()` must iterate `spec.charts` array — no hardcoded chart IDs.
4. `renderKPIs()` must derive from `spec.kpis` array — no hardcoded column names.
5. `fmt()` must always be called — never hardcode currency symbols in chart formatters.
6. Every chart must call `destroyChart(id)` before rebuild.
7. All chart configs must include the datalabels block from this file.
8. `validateSpec(spec)` runs before any render.
9. No `console.log` — use `debug(msg)` wrapper with `DEBUG_MODE` flag.
10. Every function has a JSDoc comment.
11. CSS: variables only — no hex values outside `:root`.
12. Never use `innerHTML` for user-supplied data — sanitize first.
13. `dashboard.html` is the only output file.

---

## CHANGE LOG

| Date       | Change |
|------------|--------|
| 2026-05-18 | Initial CLAUDE.md — full universal dashboard engine spec |
