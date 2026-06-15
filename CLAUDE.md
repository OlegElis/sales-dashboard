# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Пульс отдела продаж — ГК ОПОРА** is a single-file sales dashboard (`dashboard.html`) for a bankruptcy law firm's sales department (БФЛ — банкротство физических лиц). It is deployed via GitHub Pages at `olegelis.github.io`.

There is no build step, no package manager, no framework. The entire application is one self-contained HTML file.

## Development

To develop: open `dashboard.html` directly in a browser, or use a local HTTP server (required for Google Sheets CSV fetch due to CORS on some setups):

```bash
python -m http.server 8080
# then open http://localhost:8080/dashboard.html
```

To deploy: commit and push to `main` — GitHub Pages serves the file automatically.

## Architecture

### Single-file structure (`dashboard.html`)

All CSS, HTML, and JavaScript live in one file in this order:

1. **`<style>`** — CSS variables, layout (`.page-grid` 2×2 CSS Grid), component styles. Uses CSS custom properties (`--bg`, `--green`, `--amber`, `--red`, `--blue`, etc.) for theming.
2. **Login screen HTML + inline `<script>`** — password check (`AveMaria`) against `sessionStorage`. Runs immediately on load, hides the screen if already authenticated.
3. **Config `<script>`** — Google Sheets constants (`SHEET_ID`, `SHEET_NAME`), plan defaults (`PLAN`, `PLAN_TOTAL`), settings persistence via `localStorage` (`SETTINGS_KEY = 'opora_settings_v3'`), salary calculation helpers.
4. **Main HTML** — view tabs, filters, stats strip, manager rating table, charts, portfolio kanban.
5. **Main `<script>`** — all application logic (data loading, rendering, modals, charts).
6. **Modal HTML** — day modal, ZP (salary) modal, settings panel injected at the bottom of `<body>`.

### Data flow

```
Google Sheets (CSV via gviz/tq endpoint)
  → parseCSV() → allRows[]
  → render() → filtered subsets → DOM + Chart.js
```

Data is loaded once on `init()`. All filtering is done client-side in memory.

### Key global state

| Variable | Purpose |
|---|---|
| `allRows` | All raw rows from Google Sheets |
| `activeYear`, `activeMonths`, `activeDay` | Current period filter (defaults to today's date) |
| `selectedMops` | Set of manager names currently shown |
| `activeMop` | Single manager isolation mode (click on name in table) |
| `allMops`, `periodMops` | All known managers / managers with data in current period |
| `settings` | Persisted plan targets and manager start dates |

### Column mapping (`COL` object)

The Google Sheet columns are mapped by index in `COL`:
- `COL.responsible` (8) — manager login/ID
- `COL.responsibleText` (9) — manager display name
- `COL.dateCreated` (17) — meeting creation date → used for conversion calculation
- `COL.datePaid` (20) — payment date → determines if a deal is "paid"
- `COL.direction` (7) — `#1`, `#2`, `#3`, `#4` (lead source channels)
- `COL.dealStage` (30) — CRM stage name (used for portfolio kanban)
- `COL.sum` (5) — deal amount

A row is considered a paid deal only if `datePaid` is non-empty (`isPaidRow()`).

### Salary calculation (`calcZP`)

Called only in single-month mode. Components:
- **Оклад**: 30,000 ₽ prorated by working days
- **Надбавка за стаж**: 2,000 ₽/year prorated, based on `settings.startDates`
- **KPI (сделки)**: 72,000 ₽ base ± 5,000/deal above plan, −4,500/deal below
- **Конв. #1**: base 18,000 ₽, threshold 15%, plan 25%
- **Конв. #2–4**: base 18,000 ₽, threshold 45%, plan 65%
- **Средний чек**: base 12,000 ₽, threshold 150k, plan 225k

### Conversion calculation (`calcConversions`)

**Period-flow definition** (per selected month): `conv = (deals paid in the month) / (meetings created in the month)`, split by direction (`#1` vs `#2–4`).
- Denominator anchored to `dateCreated` (meetings created in the period).
- Numerator anchored to `datePaid` (deals paid in the period) — so the numerator equals the period's deal count per direction.
- These are different row sets, so conversion **can exceed 100%** (many old meetings paid in a month with few new meetings).
- `calcConversionsDirect` does the same for multi-month / whole-year selections (selected months, or all 12 for a year).
- This matches the stats-strip `#N встр / сд / конв` numbers.

(Historical note: previously a 3-month cohort method — "of meetings created in current+2 prev months, how many ever got paid". Changed to period-flow per user so the numerator equals monthly deals.)

### Views

- **Статистика** — default view, shows `#page-grid` (2-column CSS grid)
- **Портфель сделок** — kanban board by CRM stage, shows `#portfolio-section`

Switch via `switchView('stats' | 'portfolio')`.

### Charts

Uses **Chart.js 4.4.1** (CDN). Two charts:
- `dynChart` — bar+line combo, monthly dynamics (deals count, avg check, total sum)
- `dailyChart` — bar+line combo, daily cumulative fact vs forecast vs plan. Clicking a bar opens the day modal.

### Modals

- **Day modal** (`openDayModal`) — meetings created + deals paid on a specific day
- **ZP modal** (`openZpModal`) — full salary breakdown + conversion detail tables (last 3 months of meetings/deals per manager)
- **Settings panel** (`openSettings`) — password-protected (`AveMaria1`), allows overriding plan targets per month

## Current Status (as of June 2026)

### Recently completed
- Print report: `printReport()` + `@media print` styles render the Статистика view as a one-page A4-portrait daily report for the CEO (interactive controls hidden, charts placed side-by-side, `beforeprint`/`afterprint` resize the canvases)
- Favicon: cardiogram SVG in browser tab
- Default period: opens on current day/month/year
- ZP modal: full conversion detail tables (meeting-by-meeting list with paid/unpaid status)
- Conversion period changed to: current month + 2 previous months
- Stats strip: deals/sum/avg check/payroll on top row; meetings by direction on second row

### Known behaviour notes
- Conversion rates can change retroactively as old deals get paid
- Settings are stored in `localStorage` — clearing browser storage resets plan targets
- Password (`AveMaria`) is stored in plaintext in the JS — this is intentional for simplicity
- `calcConversionsDirect` (used for multi-month views) does not return `detail1`/`detail24` rows — ZP modal is only available in single-month mode so this is fine
