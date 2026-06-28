# ALFRED_DASHBOARD_CONTEXT.md
_Paste this at the start of every new session before sharing any code._

---

## What This Repo Does
A single-file HTML dashboard (`index.html`) hosted on GitHub Pages. It reads transaction data from the Project Alfred Google Sheet (published as a GViz JSON endpoint) and visualises spending and income. No backend — pure client-side HTML/CSS/JS.

---

## Live URL
`https://timothycjin-cyber.github.io/alfred-dashboard/`

---

## How Data Is Read (Read Path)
```
Google Sheet (Sheet1)
  → Published GViz endpoint (no auth required, public read)
  → fetch() in index.html
  → Parsed and rendered client-side
```

**GViz URL pattern:**
```
https://docs.google.com/spreadsheets/d/19_C3gFlY7hDjGm87k3Uke63_Tgg6TQPl6xLiGZvuEis/gviz/tq?tqx=out:json&sheet=Sheet1
```

**Sheet columns (Sheet1, zero-indexed):**
| Index | Column | Notes |
|---|---|---|
| 0 | Date | Format: `Date(YYYY,M,D)` from GViz (month is 0-indexed) — parsed in JS |
| 1 | Amount (MYR) | Numeric |
| 2 | Category | String |
| 3 | Description | String |
| 4 | Source | `telegram` or `telegram-image` |
| 5 | Type | `Expense` or `Income` |

> ⚠️ Date parsing quirk: GViz returns dates as `Date(2026,5,28)` — month is **0-indexed**. JS parser adds +1 when formatting to `YYYY-MM-DD`. This is a known source of off-by-one bugs.

---

## How Data Is Written (Write Path — IN PROGRESS)
```
Dashboard (index.html)
  → fetch() POST to Google Apps Script Web App URL
  → Apps Script appends / edits / deletes row in Sheet1
```

**Apps Script:** attached to the Google Sheet via Extensions → Apps Script.
**Deployment:** Published as a Web App (Execute as: Me, Access: Anyone).
**Web App URL:** _[paste your deployed URL here when set up]_

---

## Stack
| Layer | Tool |
|---|---|
| Hosting | GitHub Pages (static) |
| Charts | Chart.js 4.4.1 (CDN) |
| Data read | GViz JSON endpoint (public, no auth) |
| Data write | Google Apps Script Web App (POST) |
| Auth for writes | Shared secret header `X-Alfred-Key` (pending) |

---

## Key JS Variables in `index.html`
| Variable | What it holds |
|---|---|
| `SHEET_ID` | Google Sheet ID |
| `SHEET_URL` | Full GViz fetch URL |
| `allRows` | All parsed rows from Sheet |
| `activeMonth` / `activeYear` | Currently selected month filter |
| `currentView` | `'home'` or `'analytics'` |
| `donutChart` / `lineChart` | Chart.js instances (destroyed and recreated on tab switch) |
| `CAT_COLORS` | Colour map keyed by category name |

---

## UI Structure
```
Header (sticky)
  → Title: "Project Alfred"
  → Month dropdown (last 6 months)
  → Refresh button

Floating Nav Pill (fixed bottom)
  → Home tab | Analytics tab
  → Sliding indicator

Home View
  → 3 metric cards: Income | Expenses | Net Balance
  → Transaction timeline grouped by date
  → Each date group has a "Spent this day" footer row

Analytics View
  → 2 metric cards: Daily Burn Rate | Forecasted EOM Spend
  → Cumulative Spend line chart (this month vs last month)
  → Expenses by Category donut chart
  → Category Breakdown horizontal bar list
```

---

## What's Done ✅
- Full read-only dashboard working (Home + Analytics tabs)
- GViz date parsing bug fixed (`Date(Y,M,D)` → correct `YYYY-MM-DD`)
- Income vs Expense classification in timeline and metrics
- Month selector (last 6 months)
- Dark mode support
- Animated number counters
- Cumulative spend line chart (this month vs last month comparison)
- Donut chart + category breakdown bars

---

## What's Pending ❌

### 1. Floating Action Button (FAB) — New Entry
- [ ] "+" button fixed to bottom-right corner
- [ ] Opens a modal/sheet with fields: Date, Amount, Category, Description, Type (Expense/Income)
- [ ] On submit → POST to Apps Script Web App → appends row to Sheet1
- [ ] On success → refresh data (`init()`)

### 2. Edit / Delete per Transaction Row
- [ ] Tap a transaction row → opens edit modal pre-filled with that row's data
- [ ] Edit: PUT/POST to Apps Script with row identifier → Apps Script finds and updates row
- [ ] Delete: POST to Apps Script with row identifier → Apps Script deletes row

### 3. Apps Script Web App
- [ ] `doPost(e)` handler with actions: `add`, `edit`, `delete`
- [ ] Secured with shared secret: check `e.parameter.key` or header `X-Alfred-Key`
- [ ] Deployed as Web App (Execute as Me, Access: Anyone)

---

## ⚠️ Known Problem: Row Identification for Edit/Delete

**The core issue:** There is no unique ID column in the sheet. Rows can only be found by matching field values.

**Why date-matching alone fails:**
- GViz returns dates as `Date(Y,M,D)` (0-indexed month)
- The sheet stores dates as `YYYY-MM-DD` strings
- Mismatches between these two formats cause the Apps Script row lookup to silently fail — this was the root cause of edit/delete not working in the previous attempt

**Two options to solve this (decide at start of session):**

| Option | Approach | Pro | Con |
|---|---|---|---|
| A | Match by `Date + Amount + Description` concat | No schema change | Fragile if duplicates exist |
| B | Add a `UID` column to Sheet1 (e.g. timestamp on append) | Reliable, clean | Requires bot (`main.py`) and Apps Script to both write UID |

> Recommended: **Option B** — add a hidden `UID` column (col G) populated by Apps Script on every `add`. Dashboard reads it from GViz (col index 6) and passes it back on edit/delete. Bot doesn't need to change.

---

## Apps Script Skeleton (to be built)
```javascript
function doPost(e) {
  const SECRET = "your-shared-secret-here";
  const params = JSON.parse(e.postData.contents);
  if (params.key !== SECRET) return response({error: "Unauthorized"});

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Sheet1");
  const action = params.action;

  if (action === "add") {
    sheet.appendRow([params.date, params.amount, params.category,
                     params.description, "dashboard", params.type]);
  } else if (action === "edit") {
    // Find row by UID (col 7) or by date+amount+desc concat
  } else if (action === "delete") {
    // Find row by UID and delete
  }
  return response({success: true});
}

function response(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

---

## How to Start a Session
1. Paste this file as your first message
2. Paste the current `index.html`
3. State what you're building today (FAB, edit modal, Apps Script, or UID column)
4. Decide: Option A or Option B for row identification before writing any code
