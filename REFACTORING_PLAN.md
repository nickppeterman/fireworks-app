# JC's Fireworks Manager — Refactoring Plan

## What exists today

**Live URL:** https://fireworks-app-production.up.railway.app/JCs_Fireworks_Manager.html

**Architecture:** Two files deployed on Railway:
- `server.py` — Python HTTP server (PIN auth, JSON data persistence, Square API proxy)
- `JCs_Fireworks_Manager.html` — Monolithic HTML file with all CSS/JS/HTML inline
- `jcs_data.json` — Persistent data stored on Railway volume

**The server is solid.** It handles auth, data save/load, and Square API proxying cleanly. The server doesn't need a rewrite — it needs the frontend to be broken apart.

**The problem is the HTML file.** It's a single massive file containing 12 tab-based sections that were built incrementally through Cowork conversations. Each feature works, but:
- No shared component structure (same table/filter patterns repeated everywhere)
- No clear workflow between related features
- Tab organization mirrors spreadsheet tabs, not business workflows
- UI/UX is inconsistent across sections
- Impossible to maintain or extend without risking breakage

---

## Business context

**JC's Fireworks** — Seasonal fireworks retail, two stands:
- Natchez, MS (Hwy 61 North)
- Vidalia, LA (across from the Dodge Store)

**Users:** Nick and his wife (owners). Staff use Square POS only.

**Seasons:** 4th of July (~June 15 – July 5) and Christmas/New Year's (~Dec 15 – Jan 2)

**Data source:** Square POS (catalog, inventory, orders, transactions)

**Key numbers:** 584 SKUs, 400+ unique items, 3 suppliers (Winco, Spirit of 76, Jake's), 25 product categories, 10% school fundraiser giveback program

---

## Current features (all 12 tabs)

| # | Tab | What it does | Status |
|---|-----|-------------|--------|
| 1 | Dashboard | Season overview cards, low stock alerts, category breakdown, top items by order qty | Works well |
| 2 | Inventory | Full item list with filters (category, vendor, location, stock level, media), edit modal, CSV export, bulk media finder | Works, very dense |
| 3 | Pricing | Margin analysis by category, suggested prices based on target margin, set prices | Works |
| 4 | Locations | Per-location inventory tables (NTZ/VID) with category filters, inline edit | Overlaps with Inventory |
| 5 | JC's Packs | Custom bundle builder, save/load packs, build (deduct from inventory) | Unique, works |
| 6 | Orders | PO management, suggested reorders, manual order logging, active PO tracking | Works |
| 7 | Sales History | Multi-view: annual revenue chart, YoY table, location split, best days, top items, growth, season averages | Works, complex |
| 8 | P & L | Fixed costs, operating costs, labor costs, cash flow, inventory order costs by season | Works |
| 9 | Staff & Hours | Hours logging, pay summaries, schedule, hours by employee, pay grades, bonus structure | Works |
| 10 | Schools | Fundraiser tracking per school/district/zone, season leaderboard, YoY payouts, school directory | Works |
| 11 | Fulfillment | (Appears empty or minimal from audit) | Unclear |
| 12 | Order Planner | (Appears empty or minimal from audit) | Unclear |

---

## Proposed UX reorganization

### Principle: Organize by job, not data type

The 12 tabs become 4 sections, each representing a job you actually do in the business. Within each section, related tools flow naturally into each other.

### Section 1: Run the stands (daily ops)

**When you use it:** During selling season, at the stand or on your phone
**What it answers:** "What's running low? Where's my stock? What do I need to move?"

Contains:
- **Dashboard** (keep as-is, it's the landing page)
- **Stock overview** — merged from Inventory + Locations. One view with location columns, filters, search. No need for separate tabs.
- **Transfers** — merged from Locations transfer tool + Fulfillment. "Move 20 Roman Candles from NTZ to VID" in one place.

### Section 2: Buy & price product (pre-season + mid-season)

**When you use it:** Before the season and when restocking mid-season
**What it answers:** "What do I need to order? From which supplier? What should I price it at?"

Contains:
- **Order from suppliers** — merged from Orders + Order Planner + suggested reorders. One workflow: see what's low → build a PO → track it.
- **Pricing & margins** — keep as-is, it works
- **JC's Packs** — keep as-is, it's unique

### Section 3: Track the money (analysis)

**When you use it:** At home, between seasons, for planning
**What it answers:** "How are we doing? How does this compare to last year?"

Contains:
- **Sales & revenue** — merged from Sales History + P&L. Revenue charts, season drill-downs, and costs in one place.
- **Schools program** — keep as-is, it's its own workflow

### Section 4: Manage the team (admin)

**When you use it:** During season for hours, pre-season for setup
**What it answers:** "Who worked when? What do I owe people?"

Contains:
- **Staff & hours** — keep as-is
- **Settings** — Square integration, PIN management (currently a modal, could be a proper settings page)

---

## Proposed code architecture

### Phase 1: Break the monolith (keep server.py, restructure frontend)

The server stays as-is. The frontend moves from one HTML file to a React app served by the same Python server after build.

```
jcs-fireworks/
├── server.py                    ← Keep exactly as-is
├── jcs_data.json               ← Keep (Railway volume)
├── public/
│   └── index.html              ← Minimal shell
├── src/
│   ├── main.jsx                ← React entry
│   ├── App.jsx                 ← Section router + layout
│   ├── stores/
│   │   ├── dataStore.js        ← Central data (replaces localStorage scattered everywhere)
│   │   └── squareStore.js      ← Square sync logic
│   ├── sections/
│   │   ├── stands/
│   │   │   ├── Dashboard.jsx
│   │   │   ├── StockOverview.jsx
│   │   │   └── Transfers.jsx
│   │   ├── buying/
│   │   │   ├── SupplierOrders.jsx
│   │   │   ├── Pricing.jsx
│   │   │   └── Packs.jsx
│   │   ├── money/
│   │   │   ├── SalesRevenue.jsx
│   │   │   └── Schools.jsx
│   │   └── team/
│   │       ├── StaffHours.jsx
│   │       └── Settings.jsx
│   ├── components/
│   │   ├── DataTable.jsx       ← Reusable filtered/sorted table
│   │   ├── CategoryFilter.jsx  ← Reusable category dropdown
│   │   ├── LocationFilter.jsx  ← Reusable NTZ/VID toggle
│   │   ├── VendorFilter.jsx
│   │   ├── StatCard.jsx
│   │   ├── EditItemModal.jsx
│   │   └── SearchBar.jsx
│   └── utils/
│       ├── api.js              ← fetch wrappers for /api/* and /sq-api/*
│       ├── format.js           ← Currency, dates, percentages
│       └── csv.js              ← CSV export helper
├── package.json
├── vite.config.js
└── Dockerfile                  ← Build React → serve from server.py
```

### Key architectural decisions

1. **Server stays Python.** It works, it's deployed, it handles auth and Square proxy. Don't touch it.

2. **React for the frontend.** The current app is already doing React-like things (state management, conditional rendering, event handlers) — it's just doing them in vanilla JS inside one file. React gives you components, which is what this app desperately needs.

3. **Shared components.** Right now every tab has its own copy of the category filter dropdown, the data table, the stat cards. One `DataTable` component with props replaces 8 copy-pasted table implementations.

4. **Central data store.** Currently data is scattered across localStorage keys and global JS variables. One store that loads from `/api/load`, saves to `/api/save`, and provides data to all components.

5. **Build step.** Vite builds the React app into static HTML/JS/CSS. The Python server serves the built files. Deploy is: build → push to Railway.

---

## Priority shift: Build for the business, not for code cleanliness

Through discussion on April 4, 2026, we identified that the FIRST thing to build is NOT a reorganized UI — it's the **ordering decision workflow**. Nick and his wife just received Winco's new pricing list for Summer 2026. The ordering process currently takes weeks of going back and forth because they lack trusted data to decide faster. This is where the app can create the most business value immediately.

The full UI refactor (12 tabs → 4 sections) is still the right long-term plan, but the immediate build should be the ordering tool — which becomes the centerpiece of the "Buy & price product" section.

### The ordering workflow tool (build this first)

**The problem:** Ordering takes weeks not because the work is hard, but because Nick and his wife don't trust the data enough to decide faster. They review 584 SKUs item by item, category by category, weighing gut feel against incomplete information.

**The insight:** Customers shop by category. So ordering is really 25 category-level decisions + item selection within each. The tool should narrow the conversation, not open it.

**Three-step workflow:**

**Step 1 — Category budget planner**
- Set total season budget
- App shows: last season spend and revenue per category, carry-over inventory value
- Suggests starting allocation based on historical performance
- Nick and wife adjust based on gut + economy + supplier availability
- Output: budget per category

**Step 2 — Category drill-down**
- For each category: items ranked by sales velocity across seasons
- Shows: current carry-over stock, last season sold, trend (up/down/flat)
- Flags: dead stock (ordered 50, sold 8), rising stars, items not available from suppliers
- Suggests order quantities based on sell-through rate + budget constraint
- Nick and wife make the call together — app has done the math
- Supports the curation decisions: "we need 4 fountain options, not 12"

**Step 3 — Supplier PO builder**
- Groups confirmed order quantities by supplier (Winco, Jake's, Spirit of 76)
- Applies new supplier pricing from uploaded price list
- Shows total cost vs budget
- Generates POs ready to submit

**Key data inputs needed:**
- Sales history by item by season (already in the app, back to 2015)
- Current inventory by location (Square sync)
- New supplier pricing list (Winco just sent theirs — needs upload/import)
- Season budget (Nick sets this)

---

## Migration plan (phased)

### Phase 0: Snapshot and safety net
- [x] GitHub repo exists: https://github.com/nickppeterman/fireworksApp (private)
- [x] Railway auto-deploys from `main` branch (22 deployments, `grateful-solace/production`)
- [x] Dockerfile: `python:3.11-slim`, copies 2 files, runs `server.py`
- [ ] Download current `jcs_data.json` from Railway as a backup
- [ ] Create a `refactor` branch — all new work goes here until ready
- [ ] The current app stays live on `main` during the entire migration

### Phase 1: Scaffold the React app
- [ ] Create `refactor` branch from `main`
- [ ] Add `package.json`, `vite.config.js` to the repo
- [ ] Update `Dockerfile` to: install Node, build React, then serve with Python
- [ ] Update `server.py` to serve from a `dist/` folder (built React output) instead of the root
- [ ] Create the 4-section navigation shell
- [ ] Wire up `/api/load`, `/api/save`, and `/sq-api/*` calls
- [ ] Build the central data store
- [ ] Create shared components: DataTable, StatCard, filters, SearchBar
- [ ] Build the Dashboard page (it's the simplest full-featured page)
- [ ] Deploy `refactor` branch to a separate Railway environment for testing
- [ ] Test: does the dashboard show the same data as the old app?

### Phase 2: Migrate "Run the stands" section
- [ ] Stock overview (merge Inventory + Locations into one view)
- [ ] Transfers (pull from Locations transfer tool)
- [ ] Edit item modal
- [ ] CSV export
- [ ] Bulk media finder
- [ ] Test against live data

### Phase 3: Migrate "Buy & price product" section
- [ ] Supplier orders (merge Orders + Order Planner)
- [ ] Pricing & margins
- [ ] JC's Packs builder
- [ ] PO generation / logging

### Phase 4: Migrate "Track the money" section
- [ ] Sales & revenue (merge Sales History + P&L)
- [ ] Charts (annual revenue, YoY)
- [ ] Schools fundraiser program

### Phase 5: Migrate "Manage the team" section
- [ ] Staff & hours
- [ ] Settings (Square integration, PIN)

### Phase 6: Polish + cutover
- [ ] Mobile-responsive pass on all sections
- [ ] Performance audit (lazy load sections, cache Square data)
- [ ] Make new app the default URL
- [ ] Keep old HTML as fallback for 1 season
- [ ] Remove old HTML after confirmed working

---

## What NOT to change

- `server.py` — it works, don't rewrite it
- The PIN-based auth system — simple and effective for two users
- The Square integration approach (browser-side token, server proxy) — it works
- The data persistence model (JSON file on Railway volume) — it works for this scale
- Any business logic that's currently correct (margins, reorder calculations, school payout math)

---

## Timeline estimate

| Phase | Effort | Can be done in |
|-------|--------|----------------|
| Phase 0: Safety net | Small | 1 session |
| Phase 1: Scaffold + Dashboard | Medium | 2-3 sessions |
| Phase 2: Stands section | Large | 3-4 sessions |
| Phase 3: Buying section | Medium | 2-3 sessions |
| Phase 4: Money section | Medium | 2-3 sessions |
| Phase 5: Team section | Small | 1-2 sessions |
| Phase 6: Polish | Medium | 2-3 sessions |

Total: ~12-18 working sessions, spread over time as you're available.

**Recommendation:** Do Phases 0-2 first. That gives you the daily-ops tools in the new clean structure. You can run both apps side by side — use the new one for daily stand operations, fall back to the old one for everything else. Then migrate the rest at whatever pace works for your schedule.

---

*Created: April 4, 2026*
*Status: Plan approved, ready to begin Phase 0*
