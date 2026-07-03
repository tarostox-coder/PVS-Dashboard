# PowerVault Aftersale Dashboard

A single-file (`index.html`), zero-build executive dashboard for PowerVault's
after-sales service business. Open the file in any modern browser — no server,
bundler, or install step. Data comes from an uploaded Excel workbook or a
synced Google Sheet; a demo dataset renders until real data is loaded.

---

## Architecture (single file, 4 isolated script blocks)

| Block | `id` | Responsibility |
|------|------|----------------|
| 1 | `universal-data-engine` | **`window.DataEngine`** — schema-agnostic Excel ingestion: parse → dynamic header detection → fuzzy column mapping → dataset classification → typed extraction → KPI derivation. No hardcoded sheet names / column indexes. |
| 2 | *(main app)* | Rendering, KPI cards, charts (Chart.js), filters, drill-downs, import/export, Google Sheets sync, theme, snapshot. |
| 3 | `dynamic-tabs-module` | **`window.DynamicTabs`** — turns any *unknown* sheet into a runtime tab (KPI / trend / breakdown / ranking / explorer). |
| 4 | `dq-audit-module` | **`window.DQAudit`** — the Data Quality & Audit Center (this upgrade). |

Each add-on module is a self-contained IIFE that reads app globals defensively
and **never mutates the canonical `DATA` contract**. A failure in any module is
caught and degraded to an inline card — it can never break the core dashboard.

---

## Data Quality & Audit Center (Tab 06)

An additive page that reuses the existing design system (cards, KPI grid,
badges, tables, health banner, drill-down modal, both themes).

### Data flow

```
runUniversalImport(arrayBuffer)                     [existing import pipeline]
  └─ DataEngine.run → { data, report }
       report.datasetTables   full-fidelity tables for each classified sheet
       report.unclassified    unknown business sheets (already existed)
       report.grids           raw sheet grids (for lookup parsing)
       report.sheetMeta       parsed @META hints (optional, generic)
  └─ DynamicTabs.rebuild(report)                     [existing]
  └─ DQAudit.ingest(report, DATA, source)            [new hook]
       ├─ collectTables()   one uniform table shape from datasetTables +
       │                    unclassified + (demo fallback) DATA
       ├─ parseLookups()    dropdown lists from the workbook's own reference
       │                    sheets (Master_Data …), mapped to canonical fields
       │                    via the engine's synonym registry
       ├─ runRules()        business-rule validation (see below)
       ├─ computeQuality()  KPIs + Data Quality Score + health state
       └─ diffSnapshot()    Create/Update/Delete vs the previous import,
                            field-level, persisted in localStorage
```

Opening tab 06 calls `DQAudit.render()` (dispatched from `renderActiveTab()`).
Heavy work runs on import; rendering is filter-driven and cheap.

### Sections

- **Data Quality Dashboard** — 10 KPIs (Total Records, Missing Required Fields,
  Duplicate Records, Invalid Values, Invalid Dates, Invalid Dropdown Values,
  Expired Contracts, Expired Permits, Error Rate, Data Quality Score), each
  drill-down enabled, with Healthy / Warning / Critical indicators.
- **Business Rule Validation** — every issue links to its record. Rules:
  missing required fields, duplicate record id, duplicate natural key
  (contract/invoice/permit), invalid typed cells, invalid date, end-before-start,
  negative currency, invalid dropdown (validated against the workbook's own
  lists), expired contract, overdue permit.
- **Audit Center** — Date · Time · User · Sheet · Customer · Project · Record ID
  · Action · field-level Previous → New. Create / Update / Delete, expandable
  detail for multi-field changes.
- **Activity Timeline** — newest-first, grouped by date then user, drill-down.
- **Global Filters** — Date Range · User · Sheet · Customer · Project · Province
  · Action · Status. Applied consistently to KPIs, validation, audit, timeline,
  and the chart.
- **Global Search** — debounced, lazy-indexed, across records (customer /
  project / sheet / record id / key text) and the audit log.

### How the audit log works

On every import a **snapshot** (per-sheet, keyed by primary key when present,
else a content hash) is stored in `localStorage`. The next import is diffed
against it to produce Create/Update/Delete events with field-level changes.
The first import records a per-sheet baseline. Events are retained across
sessions. The log is source-driven (Excel upload or Google Sheets sync); the
"User" is resolved generically from workbook metadata (`updatedBy`/`owner`/…)
or falls back to the import source.

### localStorage keys

| Key | Contents |
|-----|----------|
| `dq_audit_events_v1` | audit event log (capped, oldest pruned) |
| `dq_snapshot_v1` | previous-import snapshot for diffing (quota-degrades gracefully: drops stored values but keeps hashes so Create/Update/Delete still detect) |

Existing keys (`gsheets_export_url`, `dashboard_src_mode`, `dashboardDebug`, …)
are untouched.

### Configuration

Tunables live in the `CFG` object at the top of the `dq-audit-module` block
(render/scan caps, score weights, health bands, search limits). No business
rule, sheet name, or column index is hardcoded — dataset structure comes from
`DataEngine._internal.DATASETS` and the workbook's own `@META` / reference sheets.

---

## Compatibility

Backward compatible with the existing Excel template, Google Sheets workflow,
import/export, dynamic sheet detection, reports, and charts. The engine changes
are purely additive (`report.datasetTables`, `report.grids`, `report.sheetMeta`,
per-column invalid-cell counters); all existing consumers see the same data.

---

## Debug mode

Append `?debug=1` to the URL (or set `localStorage.dashboardDebug = '1'`).
The last import report is exposed at `window.__dataEngineReport`; the DQ module
state at `window.DQAudit._internal.STATE`.
