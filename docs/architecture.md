# docs/architecture.md
# Component map and data flow for Universal Dashboard Engine
# Claude Code reads this before modifying any data-flow code.
# ─────────────────────────────────────────────────────────────────────────────

## Data Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│  USER ACTION                                                              │
│  Upload file (.csv / .xlsx)                                              │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  PARSE LAYER (browser, synchronous)                                      │
│  PapaParse → CSV rows[]                                                  │
│  SheetJS   → XLSX rows[]                                                 │
│  Output: RAW_DATA = Row[]   (global, immutable until next upload)        │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  COLUMN CLASSIFIER (browser JS, no network)                              │
│  Input:  RAW_DATA[0..49]                                                 │
│  Output: colSummary = {                                                  │
│    numeric:     string[]   → sum/avg KPI candidates                      │
│    date:        string[]   → time trend chart candidates                 │
│    categorical: string[]   → filter + bar/donut chart candidates         │
│    id:          string[]   → count KPI candidates, table ID column       │
│  }                                                                       │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  CLAUDE API CALL (async, ~1-3 seconds)                                   │
│  Input:  filename + colSummary + RAW_DATA[0..19]                        │
│  Model:  claude-sonnet-4-6                                               │
│  Output: DashboardSpec JSON (see schema in CLAUDE.md)                   │
│                                                                          │
│  What Claude decides:                                                    │
│  - Business domain and appropriate icons/labels                          │
│  - Which 4-6 columns become KPIs and with what aggregation              │
│  - Which 4 charts to draw and with what groupBy/valueColumn             │
│  - Which 3-4 columns become interactive filters                          │
│  - 8 data-specific insight strings with real values from sample         │
│  - valueFormat (currency / days / percentage / number)                   │
│  - Table column selection and status column positive/negative values     │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  SPEC VALIDATOR                                                          │
│  validateSpec(spec, columnNames)                                         │
│  Checks:                                                                 │
│  - All required top-level keys present                                   │
│  - spec.kpis.length >= 4                                                 │
│  - spec.charts.length >= 4                                               │
│  - All column references exist in RAW_DATA column names                 │
│  - spec.insights.length >= 6                                             │
│  On failure: throws with descriptive message → caught → showToast()     │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  FORMATTER FACTORY                                                       │
│  fmt = buildFormatter(spec)                                              │
│  Returns a function: (number) → string                                   │
│  Used by all chart datalabels, KPI values, tooltip callbacks            │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  FILTER INIT                                                             │
│  populateFilters(spec, RAW_DATA)                                         │
│  For each entry in spec.filters:                                         │
│    → Extract unique values from RAW_DATA[filter.column]                 │
│    → Build <select> element with options                                 │
│    → Append to #filtersBar                                               │
│  FILTERED_DATA = [...RAW_DATA]  (full copy, no filter active yet)       │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  renderAll(spec, data)  ←─────────────────────────────────────────────┐ │
│  ├── renderKPIs(spec, data)                                           │ │
│  │     For each kpi in spec.kpis:                                     │ │
│  │     compute value using kpi.aggregation + kpi.column               │ │
│  │     render card with icon, label, value, badge                     │ │
│  │                                                                    │ │
│  ├── renderCharts(spec, data)                                         │ │
│  │     For each chart in spec.charts:                                 │ │
│  │     destroyChart(chart.id)                                         │ │
│  │     aggregate data: groupBy + aggregation                          │ │
│  │     build Chart.js config from chart.type                         │ │
│  │     inject datalabels config from CLAUDE.md standards             │ │
│  │     new Chart(ctx, config) → store in CHARTS[chart.id]            │ │
│  │                                                                    │ │
│  ├── renderInsights(spec)                                             │ │
│  │     Render spec.insights[] as bullet cards                        │ │
│  │                                                                    │ │
│  └── renderTable(spec, data)                                          │ │
│        Sort data by spec.tableValueColumn descending                  │ │
│        Render top 10 rows using spec.tableColumns                     │ │
│        Status column → pill badge (positive/negative/neutral)        │ │
│                                                                       │ │
│  ON FILTER CHANGE:                                                    │ │
│  applyFilters() →                                                     │ │
│    read all filter <select> values                                    │ │
│    FILTERED_DATA = RAW_DATA.filter(row => all conditions match)      │ │
│    renderAll(spec, FILTERED_DATA) ────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Global State Variables

```javascript
let RAW_DATA      = [];        // Full parsed dataset, never mutated after load
let FILTERED_DATA = [];        // Current view (equals RAW_DATA when no filters active)
let CURRENT_SPEC  = null;      // DashboardSpec JSON from Claude API
let CURRENT_TITLE = '';        // Dashboard title derived from filename
const CHARTS      = {};        // Chart.js instance registry { chartId: Chart }
let fmt           = n => n;    // Number formatter, replaced by buildFormatter()
const DEBUG_MODE  = false;     // Set true locally for console output
```

---

## Aggregation Logic

```javascript
function aggregate(data, groupByCol, valueCol, method) {
  const map = {};

  data.forEach(row => {
    const key = row[groupByCol];
    if (!key) return;
    if (!map[key]) map[key] = { sum: 0, count: 0, values: [] };

    const v = parseFloat(row[valueCol]) || 0;
    map[key].sum   += v;
    map[key].count += 1;
    map[key].values.push(v);
  });

  const result = {};
  Object.entries(map).forEach(([key, bucket]) => {
    switch (method) {
      case 'sum':     result[key] = bucket.sum; break;
      case 'count':   result[key] = bucket.count; break;
      case 'average': result[key] = bucket.sum / bucket.count; break;
      case 'max':     result[key] = Math.max(...bucket.values); break;
      default:        result[key] = bucket.count;
    }
  });
  return result; // { "Category A": 4200, "Category B": 3100, ... }
}
```

---

## Chart Type Mapping

| spec.charts[i].type | Chart.js type | indexAxis | Notes |
|---------------------|---------------|-----------|-------|
| `line`              | `line`        | x         | fill:true, gradient bg, tension:0.38 |
| `bar`               | `bar`         | x         | borderRadius:8, borderSkipped:false |
| `horizontalBar`     | `bar`         | y         | indexAxis:'y', padding right 80px |
| `donut`             | `doughnut`    | —         | cutout:'62%' |
| `stackedBar`        | `bar`         | x         | stacked:true on both axes |

---

## Date Handling for Line Charts

When `spec.charts[i].groupBy` is a date column:
1. Parse all date values with `new Date()`
2. Extract month-year: `Jan 2026`, `Feb 2026`, etc.
3. Sort chronologically before rendering
4. If all dates are in same year: use month abbreviations only (Jan, Feb…)
5. If dates span multiple years: use `Mon YY` format (Jan 26, Feb 26…)

---

## KPI Aggregation by `spec.kpis[i].aggregation`

| aggregation | computation |
|-------------|-------------|
| `count`     | `data.length` |
| `sum`       | `data.reduce((s,r) => s + (+r[kpi.column] \|\| 0), 0)` |
| `average`   | sum / count |
| `ratio`     | `(data.filter(r => r[kpi.column] === kpi.ratioValue).length / data.length) * 100` |
| `unique`    | `new Set(data.map(r => r[kpi.column])).size` |
| `max`       | `Math.max(...data.map(r => +r[kpi.column] \|\| 0))` |

---

## Spec Validation Rules

`validateSpec(spec, columnNames)` must throw a descriptive Error if:

1. `spec.kpis` is empty or missing
2. `spec.charts` has fewer than 3 entries
3. `spec.insights` has fewer than 5 entries
4. Any `kpi.column` value does not exist in `columnNames` array
5. Any `chart.groupBy` or `chart.valueColumn` does not exist in `columnNames`
6. Any `filter.column` does not exist in `columnNames`
7. `spec.tableStatusColumn` is defined but not in `columnNames`

On validation error, catch in the calling code and call `showToast(err.message, 'error')`.
