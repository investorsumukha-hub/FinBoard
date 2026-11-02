# FinBoard — Implementation Prompt for Claude Code

> Hand this entire document to Claude Code as the task description. It is self-contained.

---

## 0. Your role and how to work

You are building **FinBoard**, a personal, single-user, one-stop finance dashboard for an Indian retail investor/trader. The owner has already approved the design below. Implement it **phase by phase** (section 12), committing after each phase with a clear message. Do not skip ahead; each phase must build, lint and pass tests before the next starts.

Ground rules:

1. **Discovery first (section 2).** The owner already has EOD market data, an update script, and one strategy. You must inspect those before writing the data layer. If anything in section 2 cannot be answered from files in the repo or the data folder, **stop and ask the owner** — do not guess file formats.
2. **Never modify, move, or delete the owner's existing data files or scripts.** FinBoard reads market data; only the owner's own script writes it. FinBoard may write only to its own folders (`signals/`, `cache/`, `db/`, `fundamentals/` uploads) under the configured data directory.
3. **Declarative architecture is the core requirement** (section 4). Adding a new technical strategy or valuation model must take exactly: **one entry in `finboard.config.ts` + one module file**. No edits to routing, navigation, dashboards, pipelines or registries. Treat any design that violates this as a bug.
4. **v1 runs locally** (`npm run dev` / `npm run build && npm start` on the owner's machine). Code must still be structured so that v2 can deploy to **Vercel** by swapping storage adapters and flipping env flags — no Vercel-hostile assumptions outside the adapter layer (filesystem access, child processes and PGlite are confined to clearly named `local` adapters).
5. Everything is **India-only**: NSE symbols, INR, Indian number formatting (`₹1,23,45,678.90`, lakhs/crores toggles), IST dates, Indian FY (Apr–Mar).
6. Prefer boring, well-maintained libraries. Keep modules small and typed. No `any` in public interfaces.
7. All computations on market data must be **pure and look-ahead free** (a function evaluated at bar `t` may only read bars `≤ t`), because a backtesting engine will reuse them in v2.

---

## 1. Product summary

A dark-mode-first web app with a sidebar whose hierarchy is defined entirely by a config file:

```
Dashboard                        net-worth home page
Money
  ├─ Accounts                    bank, cash, wallets
  ├─ Transactions                income/expenses, bank CSV import, categorisation
  ├─ Budgets                     monthly per category
  └─ Liabilities                 loans, credit cards, EMIs
Investments
  ├─ Overview                    allocation + returns across all asset classes
  ├─ Stocks                      synced from broker APIs (Zerodha first), file import as fallback
  ├─ Mutual Funds                synced from broker APIs (Zerodha Coin) + AMFI NAVs
  ├─ Fixed Income                FD, RD, PPF, EPF, NPS, bonds
  └─ Other Assets                gold/SGB, real estate, crypto, misc
Trading Journal
  ├─ Trades                      auto-built from tradebook + manual notes/tags
  └─ Analytics                   equity curve, win rate, expectancy, R-multiples
Research & Insights
  ├─ Fundamental  (dashboard)
  │    ├─ Relative Valuation
  │    ├─ Asset-Based Valuation
  │    └─ Income-Based Valuation
  └─ Technical    (dashboard)
       ├─ VCP
       ├─ SEPA
       ├─ Darvas Box
       ├─ ABC
       ├─ 44 MA
       └─ <owner's existing strategy, ported>
Planning (phase 9)
  ├─ Goals
  ├─ Tax (capital gains)
  └─ Insurance
Settings                         data paths, EOD update, broker connections, imports, categories, preferences
```

**Broker connections** (Settings → Connections) pull holdings, positions, trades, orders, funds and MF data directly from broker APIs. Zerodha Kite Connect is the first connector, and the design is broker-agnostic (§8A).

A global **"Update EOD"** button (top bar) runs the owner's existing update script, streams its output, then re-runs all strategy scans.

---

## 2. Discovery (do this before Phase 2; ask the owner for anything missing)

Collect and record the answers in `docs/DATA_CONTRACT.md` (create it). This document becomes the contract the data adapters implement.

1. **Data directory**: absolute path(s) of the owner's EOD data. Will be supplied via `.env.local` as `FINBOARD_EOD_DIR`. FinBoard's own writable folder is `FINBOARD_DATA_DIR` (may be the same parent).
2. **File format** of the per-stock EOD files: CSV or Excel (`.xlsx`)? One file per symbol? Naming convention (e.g. `RELIANCE.csv`, `RELIANCE.NS.xlsx`)? Sheet name if Excel?
3. **Columns**: exact header names, order, date format, number formats, whether delivery qty/% exists, whether prices are raw or already split/bonus-adjusted, sort order (ascending/descending), duplicate handling.
4. **Index data**: is there Nifty 50 / Nifty 500 / sector index history? Where and in what format? (Needed for RS rating, beta and market-trend widgets; if missing, these features degrade gracefully and say so in the UI.)
5. **Universe**: the strategy scan universe is **every stock that has a file in the owner's EOD data folder** (the stocks "represented in the Excel files"). Confirm this interpretation with the owner. Make the universe source pluggable (`universe: { source: "eod-files" | "list-file" | "fundamentals-files" }` in config) so it can change without code edits.
6. **Update script**: path, language/runtime (e.g. `python update.py`), working directory, required env/venv, expected runtime, how it signals success/failure (exit code? log lines?), whether it is safe to run while FinBoard is reading files.
7. **Existing strategy**: its source file, exact rules, parameters and output format. It will be **ported** to a FinBoard strategy module (section 6) and verified for parity (section 11).
8. **Corporate actions**: does the owner's data already adjust for splits/bonuses? If not, is there a corporate-actions file? (If neither, implement adjustment as a pluggable step and leave it disabled with a visible warning.)
9. Ask for **one or two sample files** of each kind (EOD stock file, index file, Screener export, Zerodha tradebook, Zerodha holdings, a bank statement CSV) and place anonymised copies under `test/fixtures/` for parser tests.
10. **Broker API access** (needed before Phase 5): confirm the owner has created a Kite Connect app, which plan it is on, and that the redirect URL in §8A.5 is registered. Never ask the owner to paste the API secret into chat or commit it — they enter it in the Connections UI. For connector tests, record **anonymised** API responses as fixtures (`test/fixtures/brokers/zerodha/*.json`); never commit real tokens, client IDs or account numbers.

Do **not** build NSE downloaders/backfill in v1 — the owner's script owns data acquisition. Design the `MarketStore` so an NSE-fetching adapter could be added later without touching consumers.

---

## 3. Tech stack

| Concern | Choice | Notes |
|---|---|---|
| Framework | Latest stable **Next.js (App Router)** + **TypeScript (strict)** | Server Components by default; client components only where interactive |
| Styling/UI | **Tailwind CSS** + **shadcn/ui** + `lucide-react` icons | Dark mode default, light mode toggle via `next-themes` |
| Charts | **Recharts** (analytics) + **TradingView `lightweight-charts`** (candles, overlays, markers, boxes) | |
| Tables | `@tanstack/react-table` | Sorting, filtering, column visibility, CSV export |
| Forms/validation | `react-hook-form` + **Zod** | Zod schemas also auto-generate strategy/valuation param forms |
| Personal DB | **Drizzle ORM** with **PGlite** (`drizzle-orm/pglite`) in v1, **Neon** (`drizzle-orm/neon-http`) in v2 | One Postgres schema for both. Add `@electric-sql/pglite` to `serverExternalPackages` in `next.config` |
| Excel | SheetJS (`xlsx`, installed from the official SheetJS CDN tarball, not the stale npm version) or `exceljs` | For Screener exports and Excel EOD files |
| CSV | `papaparse` | Streaming parse for large files |
| Auth | **Auth.js** with Google provider, single allowed email | Controlled by `AUTH_MODE=none|google`; v1 default `none` |
| Dates | `date-fns` + `date-fns-tz` (Asia/Kolkata) | |
| Money math | Integer paise or `decimal.js` for ledger amounts; never float sums for balances | |
| Tests | **Vitest** (unit), **Playwright** (a few smoke e2e; Chromium already available) | |
| Lint/format | ESLint + Prettier | `npm run lint`, `npm run typecheck`, `npm test` must pass |

---

## 4. Declarative architecture (core requirement)

### 4.1 `finboard.config.ts` (repo root)

Single source of truth for navigation hierarchy, page types and module wiring. Typed by `defineConfig`. Arbitrary nesting depth.

```ts
import { defineConfig } from "@/core/config";

export default defineConfig({
  app: { name: "FinBoard", currency: "INR", timezone: "Asia/Kolkata" },
  universe: { source: "eod-files" },          // see §2.5
  nav: [
    { id: "dashboard", title: "Dashboard", icon: "LayoutDashboard", page: "home" },
    {
      id: "money", title: "Money", icon: "Wallet",
      children: [
        { id: "accounts",     title: "Accounts",     page: "money/accounts" },
        { id: "transactions", title: "Transactions", page: "money/transactions" },
        { id: "budgets",      title: "Budgets",      page: "money/budgets" },
        { id: "liabilities",  title: "Liabilities",  page: "money/liabilities" },
      ],
    },
    // investments, journal ...
    {
      id: "research", title: "Research & Insights", icon: "Microscope",
      children: [
        {
          id: "fundamental", title: "Fundamental", page: "fundamental-dashboard",
          children: [
            { id: "relative", title: "Relative Valuation", approach: "relative", models: [
              () => import("@/modules/valuation/relative/pe"),
              () => import("@/modules/valuation/relative/pb"),
              () => import("@/modules/valuation/relative/ev-ebitda"),
              () => import("@/modules/valuation/relative/ps"),
              () => import("@/modules/valuation/relative/peg"),
            ]},
            { id: "asset", title: "Asset-Based Valuation", approach: "asset", models: [
              () => import("@/modules/valuation/asset/book-value"),
              () => import("@/modules/valuation/asset/adjusted-nav"),
              () => import("@/modules/valuation/asset/liquidation"),
              () => import("@/modules/valuation/asset/graham-number"),
            ]},
            { id: "income", title: "Income-Based Valuation", approach: "income", models: [
              () => import("@/modules/valuation/income/dcf-fcff"),
              () => import("@/modules/valuation/income/dcf-fcfe"),
              () => import("@/modules/valuation/income/ddm"),
              () => import("@/modules/valuation/income/residual-income"),
              () => import("@/modules/valuation/income/epv"),
            ]},
          ],
        },
        {
          id: "technical", title: "Technical", page: "technical-dashboard",
          children: [
            { id: "vcp",    title: "VCP",        strategy: () => import("@/modules/technical/vcp") },
            { id: "sepa",   title: "SEPA",       strategy: () => import("@/modules/technical/sepa") },
            { id: "darvas", title: "Darvas Box", strategy: () => import("@/modules/technical/darvas") },
            { id: "abc",    title: "ABC",        strategy: () => import("@/modules/technical/abc") },
            { id: "ma44",   title: "44 MA",      strategy: () => import("@/modules/technical/ma44") },
            // { id: "<owner>", title: "<Owner strategy>", strategy: () => import("@/modules/technical/<owner>") },
          ],
        },
      ],
    },
    {
      id: "settings", title: "Settings", icon: "Settings", page: "settings",
      children: [
        { id: "connections", title: "Connections", page: "settings/connections" },
        // data, fundamentals, categories, preferences ...
      ],
    },
  ],
  brokers: [                                   // §8A — one entry per broker connector
    { id: "zerodha", connector: () => import("@/modules/brokers/zerodha") },
  ],
});
```

Node kinds (discriminated union, validated with Zod at startup — fail fast with a readable error):

| Field | Meaning |
|---|---|
| `page: "<key>"` | Render a built-in page from the page registry (`src/core/pages/registry.ts`, a static map of page keys → components). Built-in pages are the only things in a registry; strategies/models are never registered anywhere except the config. |
| `strategy: () => import(...)` | Render the generic `StrategyPage` for that module. |
| `approach` + `models[]` | Render the generic `ApproachPage` listing those valuation models. |
| `children` only | Group node; its URL redirects to the first child (or renders an auto-generated index of children). |
| `component: () => import(...)` (optional on any node) | Escape hatch: custom React component replaces the generic renderer. |
| `hidden`, `badge`, `icon`, `description` | Optional presentation fields. |

URL = path of `id`s, e.g. `/research/technical/abc`.

### 4.2 Resolution and rendering

- `src/core/config/resolve.ts` — walks the tree, builds `{ path → node }`, breadcrumb trail, and a **serializable nav tree** (no functions) for the client sidebar.
- `app/(app)/[[...slug]]/page.tsx` — the **only** route for config-driven pages. It resolves the node and renders the appropriate renderer. `generateStaticParams` is not required in v1.
- Config and module loaders are **server-only** (`import "server-only"`). The client never imports module code; it receives plain data from server components / route handlers.
- **Dashboards discover children from config.** `TechnicalDashboard` iterates over its sibling/child strategy nodes; `FundamentalDashboard` iterates over approach nodes and their models. A new strategy automatically gets a dashboard card, a sidebar entry, a page, and inclusion in the scan pipeline.
- A test (`core/config/config.test.ts`) asserts: every node id is unique among siblings, every `page` key exists in the registry, every strategy/model module loads and satisfies its schema, and every module `id` matches its node `id`.

### 4.3 Strategy module contract

`src/core/strategy/define.ts`:

```ts
export interface Bar { date: string; open: number; high: number; low: number; close: number; volume: number; deliveryPct?: number }
export interface Series { symbol: string; bars: Bar[] }          // ascending by date, adjusted per settings

export interface StrategyContext {
  index?: (name: string) => Series | undefined;                  // e.g. "NIFTY 50"
  rsRating?: (symbol: string) => number | undefined;             // 1–99, precomputed per scan date
  asOf: string;                                                  // scan date
}

export interface StrategyHit {
  score: number;               // 0–100, used for ranking
  entry?: number; stop?: number; target?: number;
  meta: Record<string, unknown>; // values for custom columns
  notes?: string[];            // human-readable reasons ("SMA50 rising 5d", "Lower BB 0.6% from SMA50")
}

export type Overlay =
  | { type: "line"; id: string; label: string; data: { time: string; value: number }[] }
  | { type: "band"; id: string; label: string; upper: ...; lower: ... }
  | { type: "box";  id: string; label: string; from: string; to: string; top: number; bottom: number }
  | { type: "marker"; time: string; position: "above" | "below"; shape: "arrowUp" | "arrowDown" | "circle"; text?: string }
  | { type: "hline"; id: string; label: string; price: number };

export function defineStrategy<P extends z.ZodObject<any>>(s: {
  id: string;
  name: string;
  description: string;           // markdown, shown on the strategy page ("Rules" panel)
  params: P;                     // Zod schema WITH defaults; drives auto-generated form
  lookback: number;              // minimum bars required
  scan(series: Series, params: z.infer<P>, ctx: StrategyContext): StrategyHit | null;  // evaluate at last bar only
  overlays(series: Series, params: z.infer<P>, ctx: StrategyContext): Overlay[];
  columns?: { key: string; label: string; format: "number" | "pct" | "inr" | "date" | "text" }[];
  needs?: ("index" | "rsRating" | "deliveryPct")[];               // pipeline precomputes only what's needed
}): Strategy<P>;
```

`scan` must only use `series.bars` up to the last element — the pipeline will later call it on truncated series for backtesting.

### 4.4 Valuation model contract

`src/core/valuation/define.ts`:

```ts
export function defineValuationModel<I extends z.ZodObject<any>>(m: {
  id: string;
  name: string;
  approach: "relative" | "asset" | "income";
  description: string;                       // markdown incl. formula
  inputs: I;                                 // Zod schema with defaults (e.g. WACC, growth, haircuts)
  defaults?(f: Financials, mkt: MarketSnapshot): Partial<z.infer<I>>;   // data-derived defaults (e.g. beta, historical growth)
  compute(f: Financials, mkt: MarketSnapshot, inputs: z.infer<I>, peers?: PeerSet): {
    fairValuePerShare: number | null;        // null + reason when not applicable (e.g. DDM with no dividends)
    reason?: string;
    breakdown: { label: string; value: number; unit: "inr_cr" | "inr" | "pct" | "x" | "count" }[];
    sensitivity?: { rowLabel: string; colLabel: string; rows: number[]; cols: number[]; values: (number | null)[][] };
    warnings?: string[];
  };
}): ValuationModel<I>;
```

---

## 5. Data layer

### 5.1 Interfaces (`src/data/market/types.ts`)

```ts
interface MarketStore {
  listSymbols(): Promise<string[]>;                         // universe per config
  getSeries(symbol: string, opts?: { from?: string; to?: string; adjusted?: boolean }): Promise<Series>;
  getIndex(name: string, opts?: ...): Promise<Series | undefined>;
  lastDate(): Promise<string | undefined>;                  // latest trading date available
  latestClose(symbol: string): Promise<{ date: string; close: number } | undefined>;
}
interface SignalStore { write(date: string, results: ScanResults): Promise<void>; read(date?: string): Promise<ScanResults | undefined>; listDates(): Promise<string[]> }
interface FundamentalsStore { list(): Promise<FundamentalsMeta[]>; get(symbol: string): Promise<Financials | undefined>; put(symbol: string, file: Buffer): Promise<void> }
interface EodUpdater { run(onLog: (line: string) => void, signal: AbortSignal): Promise<{ ok: boolean; exitCode: number; durationMs: number }> }
```

### 5.2 v1 adapters (`src/data/market/local/`)

- `LocalFileMarketStore`: reads the owner's files per `docs/DATA_CONTRACT.md`. Column mapping, date format, file pattern, and sheet name come from a typed **`eod` section in config or `.env.local`** (not hard-coded), e.g.

  ```ts
  eod: { dir: env.FINBOARD_EOD_DIR, pattern: "{symbol}.csv", format: "csv",
         columns: { date: "Date", open: "Open", high: "High", low: "Low", close: "Close", volume: "Volume" },
         dateFormat: "yyyy-MM-dd", adjusted: true }
  ```
- In-memory LRU cache of parsed series keyed by `(path, mtimeMs, size)`; invalidated automatically after an update run.
- Normalise: sort ascending, drop exact-duplicate dates (keep last), drop rows with non-positive prices, log anomalies once.
- If data is not adjusted and a corporate-actions source exists, apply back-adjustment factors at read time (`adjusted: true` option); never rewrite files.
- `LocalSignalStore`: `FINBOARD_DATA_DIR/signals/YYYY-MM-DD.json` (+ `latest.json` symlink/copy).
- `LocalFundamentalsStore`: `FINBOARD_DATA_DIR/fundamentals/{SYMBOL}.xlsx` + parsed cache `fundamentals/.cache/{SYMBOL}.json` (invalidate on mtime).
- `ScriptEodUpdater`: spawns the owner's command (`EOD_UPDATE_COMMAND`, `EOD_UPDATE_CWD` from `.env.local`) with `child_process.spawn` (no shell interpolation of user input), streams stdout/stderr lines, enforces a single concurrent run via a lock file, supports cancel. **Only enabled when `FINBOARD_MODE=local`.**

### 5.3 Update flow

1. User clicks **Update EOD** (top bar; also on Settings and Technical dashboard).
2. `POST /api/eod/update` starts a job; `GET /api/eod/update/stream` is an **SSE** stream of `{ phase: "script" | "scan" | "broker-sync" | "snapshot", line?, progress?, done?, error? }`.
3. Phase `script`: run `ScriptEodUpdater`. If it fails, stop and show the tail of the log.
4. Phase `scan`: invalidate caches, compute shared precomputations (RS ratings, index series) once, then run **every strategy found in the config** over the universe → `SignalStore.write(lastDate, ...)`. Show per-strategy progress and hit counts. Parallelise over symbols with a bounded worker pool (`p-limit`); total should finish in seconds-to-a-minute for ~2,000 symbols.
5. Phase `broker-sync`: run the broker sync engine (§8A.4) for every connection with a valid session; expired connections are skipped with a "Reconnect" notice (never fail the whole update).
6. Phase `snapshot`: write a net-worth snapshot (section 8) and refresh mutual-fund NAVs (AMFI `NAVAll.txt`, public; tolerate failure).
7. UI shows "Last updated: <date>, <time> IST" from the latest signals file.

Also expose the same flow as CLI: `npm run eod:update` (script + scan + snapshot) and `npm run scan [-- --date YYYY-MM-DD --strategy abc]` (scan only).

### 5.4 v2 seams (do not implement now; keep the seams)

`BlobMarketStore`/`S3MarketStore`, Neon DB, `AUTH_MODE=google`, scheduled updater, DuckDB-based backtester calling `strategy.scan` on truncated series. Choose implementations via a factory in `src/data/index.ts` driven by env vars.

---

## 6. Technical strategies (v1)

All parameters are Zod fields with the defaults below and are editable from the strategy page (re-scan on demand for the current date via `POST /api/scan` with params; saved param presets are stored in the DB). Common helpers live in `src/lib/indicators/` (SMA, EMA, Bollinger, ATR, RSI, rolling high/low, swing-pivot detection, linear slope) with unit tests against hand-computed fixtures.

**Bullish candle helper** (shared): `close > open` and `(close − low) / (high − low) ≥ minClosePosition` (default 0.5); optional `close > prevHigh` (default off).

### 6.1 ABC — Siddharth Bhanushali (owner-confirmed defaults)
- **A — rising MA**: `SMA(close, maPeriod=50)` rising: `SMA[t] > SMA[t − slopeLookback]` with `slopeLookback = 5` (optionally require monotonic rise over the window; default off).
- **B — lower Bollinger Band touching/near the MA**: `BB(period=20, stdDev=2)`; `|lowerBB[t] − SMA50[t]| / SMA50[t] ≤ nearTolerancePct = 1%`.
- **C — bullish candle** on bar `t` (helper above).
- Output: `entry = high[t]` (buy above), `stop = low[t]`, `target = entry + rr × (entry − stop)` with `rr = 2`. Score: higher when B distance is smaller, slope steeper, candle body stronger.
- Overlays: SMA50 line, Bollinger bands, marker on C, hlines for entry/stop/target.
- Columns: SMA slope %, lowerBB–SMA distance %, candle close position.

### 6.2 44 MA — Siddharth Bhanushali
- `SMA(close, 44)` rising over `slopeLookback = 5` bars.
- Support touch: `low[t] ≤ SMA44[t] × (1 + touchTolerancePct = 1%)` and `close[t] > SMA44[t]`.
- Bullish candle on `t` (helper above).
- `entry = high[t]`, `stop = low[t]`, `target` at `rr = 2`.
- Overlays: SMA44, marker, entry/stop/target lines.

### 6.3 SEPA — Minervini Trend Template
All must hold at `t`:
1. `close > SMA150` and `close > SMA200`
2. `SMA150 > SMA200`
3. `SMA200` rising for at least `sma200RisingDays = 22` bars (`SMA200[t] > SMA200[t−22]`)
4. `SMA50 > SMA150` and `SMA50 > SMA200`
5. `close > SMA50`
6. `close ≥ 1.30 × 52-week low` (`minAboveLowPct = 30`)
7. `close ≥ 0.75 × 52-week high` (`maxBelowHighPct = 25`)
8. `RS rating ≥ minRs = 70`

**RS rating** (precomputed once per scan date across the universe): `perf = 0.4·r(63) + 0.2·r(126) + 0.2·r(189) + 0.2·r(252)` where `r(n)` is the n-bar return; percentile rank → 1–99. Note in the UI that RS is relative to the configured universe. Score = number of criteria met + RS bonus; also show near-misses (7/8) in a separate tab.

### 6.4 VCP — Volatility Contraction Pattern
- Pre-condition: SEPA template passes (param `requireTrendTemplate = true`).
- Within the last `baseMaxBars = 65` bars, detect swing highs/lows using a pivot window (`pivotWindow = 5`). Build a sequence of contractions (pullback from a swing high to the next swing low): count `minContractions = 2` … `maxContractions = 6`, each pullback depth ≤ previous × `contractionRatio = 0.8` (allow small tolerance), first contraction depth ≤ `maxFirstDepthPct = 35`, last ≤ `maxLastDepthPct = 10`.
- Volume dry-up: average volume over the last contraction < `volumeDryUp = 0.7 ×` 50-day average volume.
- Pivot = high of the last contraction; flag `status: "setup"` if `close < pivot` and within `pivotProximityPct = 5`, `status: "breakout"` if `close > pivot` today with volume ≥ 1.4× 50-day average.
- Overlays: contraction trendlines/boxes labelled with depth %, pivot hline, breakout marker.
- Columns: #contractions, depths (e.g. "24% → 11% → 5%"), volume dry-up ratio, distance to pivot %.

### 6.5 Darvas Box
- Trigger: new `highLookback = 252`-bar high.
- Box top: the high that is not exceeded for the next `confirmBars = 3` bars. Box bottom: the lowest low after the top that is not undercut for the next `confirmBars = 3` bars.
- Signal `breakout` when `close > boxTop` with volume ≥ `volumeMultiplier = 1.5×` 50-day avg; `stop = boxBottom`. Track box stacking (up to last 5 boxes) for overlays.
- Columns: box top/bottom, box height %, days in box, status (forming/confirmed/breakout).

### 6.6 Owner's existing strategy
Port it into `src/modules/technical/<id>.ts` using the contract, keep its rules exactly, and add a **parity test** (section 11). Add its entry to the config.

### 6.7 Strategy page (generic `StrategyPage`)
- Header: name, description (rules rendered from markdown), last scan date, hit count.
- Params panel (auto-generated from Zod) with "Run scan" + "Save preset" + "Reset defaults".
- Screener table: symbol, name, close, change %, score, entry/stop/target, risk %, custom columns, "in watchlist" / "in holdings" badges, "has fundamentals" badge. Sort, filter, export CSV.
- Row click → chart drawer: candles + volume (+ delivery % if available), strategy overlays, buttons "Add to watchlist", "Log planned trade" (prefills journal), "Open fundamentals".
- History tab: hits for this strategy over previous scan dates (from `signals/`).

### 6.8 Technical dashboard (generic `TechnicalDashboard`)
- Market strip (if index data exists): Nifty 50 close, vs SMA50/200, % of universe above SMA50/200 (breadth), advance/decline for last date.
- One card per strategy node (auto-discovered): today's hits, new vs yesterday, top 5 by score, link to page.
- "Confluence" table: symbols hit by ≥ 2 strategies.
- "Your stocks" table: holdings & watchlist symbols with which strategies hit today.

---

## 7. Fundamental research

### 7.1 Screener.in Excel import
- Upload page (Settings → Fundamentals and a button on the Fundamental dashboard): accepts one or many `.xlsx` files exported from Screener.in; infer symbol from file (company name/NSE code inside "Data Sheet") with a confirmation step allowing the owner to set/override the NSE symbol. Also pick up files dropped directly into `FINBOARD_DATA_DIR/fundamentals/`.
- Parse the **"Data Sheet"** sheet by **searching row labels**, never fixed cell addresses. Sections and labels (verify against the fixture; tolerate minor label variations via an alias map):
  - META: Company Name, Number of shares, Face Value, Current Price, Market Capitalization
  - PROFIT & LOSS (annual, ~10 years; header row "Report Date"): Sales, Raw Material Cost, Change in Inventory, Power and Fuel, Other Mfr. Exp, Employee Cost, Selling and admin, Other Expenses, Other Income, Depreciation, Interest, Profit before tax, Tax, Net profit, Dividend Amount
  - Quarters: Sales, Expenses, Other Income, Depreciation, Interest, Profit before tax, Tax, Net profit, Operating Profit
  - BALANCE SHEET: Equity Share Capital, Reserves, Borrowings, Other Liabilities, Total, Net Block, Capital Work in Progress, Investments, Other Assets, Receivables, Inventory, Cash & Bank, No. of Equity Shares, New Bonus Shares, Face value
  - CASH FLOW: Cash from Operating Activity, Cash from Investing Activity, Cash from Financing Activity, Net Cash Flow
  - PRICE row; DERIVED: Adjusted Equity Shares in Cr
- Units: Screener amounts are **₹ crore**; convert carefully; per-share values use adjusted shares. Store a normalised `Financials` object (typed, per fiscal year + quarters) in the JSON cache. Show a parse report (found/missing fields, years covered) after upload.
- Derived metrics (`src/lib/fundamentals/derive.ts`): EBITDA, EBIT, EPS, BVPS, DPS, payout, ROE, ROCE, debt/equity, net debt, interest coverage, margins, working capital, **capex ≈ ΔNet Block + ΔCWIP + Depreciation** (flag as approximation), FCFF ≈ CFO − capex, FCFE ≈ FCFF − interest×(1−t) + net borrowing, 3/5/10-yr CAGRs for sales/profit/EPS/BV.
- **Market data always comes from the EOD store** (latest close) so valuations refresh daily; Screener's "Current Price" is used only as a fallback with a warning.

### 7.2 Valuation models (defaults editable per company, saved to DB as `valuation_runs`)

Relative (peer set = other uploaded companies in the same industry, editable; **fallback / additional view: the company's own 5- and 10-year median multiple** — label clearly which basis is used):
- **P/E**: fair = median peer (or historical) P/E × TTM EPS.
- **P/B**: fair = median P/B × BVPS.
- **EV/EBITDA**: fair EV = median multiple × EBITDA → equity = EV − net debt → per share.
- **P/S**: median P/S × sales per share.
- **PEG**: fair P/E = target PEG (default 1.0) × expected EPS growth % (default: 5-yr EPS CAGR, capped).

Asset-based:
- **Book value**: BVPS.
- **Adjusted NAV**: editable adjustments per balance-sheet line (e.g. investments at market, receivables provision).
- **Liquidation value**: haircuts — cash 100%, investments 80%, receivables 80%, inventory 50%, net block 40%, CWIP 20%, other assets 30% — minus all liabilities (borrowings + other liabilities).
- **Graham number**: √(22.5 × EPS × BVPS) (n/a if EPS or BVPS ≤ 0).

Income-based:
- **DCF (FCFF)**: 2-stage. Inputs: base FCFF (avg of last 3 yrs by default), stage-1 growth (default: min(5-yr sales CAGR, 20%)), stage-1 years 5 (option 10 with fade), terminal growth 5%, WACC from Rf 7.0%, ERP 6.5%, beta (computed vs Nifty 50 weekly returns over 2 yrs if index data exists, else 1.0), pre-tax cost of debt = interest / avg borrowings, tax 25.17%, D/E at market. Equity = EV − net debt. Sensitivity: WACC × terminal growth grid.
- **DCF (FCFE)**: same structure discounted at cost of equity.
- **DDM**: Gordon (single stage) and two-stage; n/a with reason if no dividends.
- **Residual income**: BV₀ + Σ (EPSₜ − kₑ·BVₜ₋₁)/(1+kₑ)ᵗ + terminal.
- **EPV (Greenwald)**: normalised EBIT (avg margin × current sales) × (1−t) / WACC − net debt (+ excess cash).

Every model shows: formula (markdown), inputs form, breakdown table, sensitivity (where meaningful), warnings (negative earnings, short history, approximated capex, etc.).

### 7.3 Approach pages (generic `ApproachPage`)
Company selector (only symbols with fundamentals) → tabs/cards per model in that approach → each card: fair value per share, upside vs latest close, margin of safety, expand for breakdown/sensitivity/inputs. Approach summary: min/median/max of its models.

### 7.4 Fundamental dashboard (generic `FundamentalDashboard`)
- Company selector + "Upload Screener file" button + list of companies with fundamentals (last upload date, latest close, composite fair value, upside).
- For the selected company: **football-field chart** (horizontal range per model, grouped by approach, vertical line at current price), approach summary cards linking to sub-pages, quality snapshot (ROE, ROCE, D/E, interest coverage, sales/profit CAGRs, FCF trend sparkline), 10-year financial charts (sales, net profit, EPS, BVPS, CFO vs net profit).
- "Technical status" strip: which strategies currently hit this symbol (cross-link to technical pages).

---

## 8. Personal finance

### 8.1 DB schema (`src/data/db/schema.ts`, Drizzle pg-core; migrations in `drizzle/`)
```
accounts(id, type[bank|cash|wallet|demat|credit_card|loan], name, institution, currency='INR', opening_balance, opened_on, closed_on, notes)
categories(id, name, parent_id, kind[income|expense|transfer], color, icon)
category_rules(id, pattern, match[contains|regex], category_id, priority)
cash_transactions(id, account_id, date, amount_paise, direction[in|out], category_id, payee, description, transfer_group_id, import_batch_id, hash UNIQUE)
budgets(id, category_id, month, amount_paise)
liabilities(id, account_id, type[home|car|personal|education|gold|credit_card|other], lender, principal_paise, rate_pct, tenure_months, emi_paise, start_date, emi_day, notes)
instruments(id, asset_class[equity|mf|etf|fd|rd|ppf|epf|nps|bond|gold|sgb|real_estate|crypto|other], symbol, isin, scheme_code, name)
investment_txns(id, instrument_id, account_id, connection_id NULL, date, traded_at, type[BUY|SELL|DIVIDEND|INTEREST|SPLIT|BONUS|DEPOSIT|WITHDRAWAL], quantity, price, amount_paise, fees_paise, fees_estimated bool, origin[api|file|manual], external_id, dedupe_hash, import_batch_id, UNIQUE(connection_id, external_id))
asset_valuations(id, instrument_id, date, value_paise)          -- for FD/PPF/EPF/NPS/real estate/other manual marks
fixed_deposits(id, instrument_id, principal_paise, rate_pct, compounding, start_date, maturity_date, payout)
trades(id, symbol, side[LONG|SHORT], entry_date, entry_price, exit_date, exit_price, quantity, stop_loss, target, fees_paise, setup (strategy id or free text), emotion, rating, notes, screenshot_path, status[planned|open|closed], source_trade_ids[])
watchlist(id, symbol, added_on, notes, tags[])
alerts(id, symbol, rule jsonb, active, last_triggered_on)
strategy_presets(id, strategy_id, name, params jsonb)
valuation_runs(id, symbol, model_id, inputs jsonb, fair_value, created_at)
net_worth_snapshots(date PK, assets_paise, liabilities_paise, breakdown jsonb)
goals(id, name, target_paise, target_date, linked_instrument_ids[], priority)
insurance_policies(id, type, insurer, policy_no, sum_assured_paise, premium_paise, frequency, next_due, nominee, notes)
import_batches(id, kind, connection_id, importer_id, filename, imported_at, rows, status)
settings(key PK, value jsonb)

-- broker connectors (§8A)
broker_connections(id, broker_id, label, client_id, account_id → accounts, status[connected|expired|error|disconnected], credentials_enc, session_enc, session_expires_at, last_sync_at, created_at)
broker_sync_runs(id, connection_id, trigger[manual|login|eod|cli], started_at, finished_at, status, counts jsonb, error)
broker_sync_coverage(connection_id, kind, date, PRIMARY KEY(connection_id, kind, date))
broker_raw_records(id, connection_id, kind, external_id, payload jsonb, fetched_at, UNIQUE(connection_id, kind, external_id))
broker_holdings_snapshots(connection_id, date, isin, symbol, quantity, average_price, last_price, kind[equity|mf], PRIMARY KEY(connection_id, date, isin))
```

### 8.2 Broker data
Broker data (holdings, positions, trades, orders, funds, MF holdings) comes from **broker connectors** — see **§8A**. There is no Zerodha-specific code outside `src/modules/brokers/zerodha/`. Everything downstream (holdings, XIRR, journal, reconciliation, dashboards) consumes only the **canonical broker models** defined in §8A.2.
- **Trade matching** into `trades` (journal): FIFO per (connection, symbol); intraday (same-day buy/sell) and swing round-trips become closed trades; open quantities become open trades. Re-running a sync/import must not duplicate trades; manual notes/tags on existing trades are preserved.
- **Reconciliation**: after every sync, compare holdings derived from `investment_txns` with the broker's own holdings snapshot (qty and avg price per ISIN). Show differences with likely causes (missing trade days, corporate action, off-market transfer, IPO allotment) and actions ("import tradebook for <date range>", "add corporate action", "create adjusting entry").
- Corporate actions (splits/bonuses) for holdings: manual entry form in v1; suggest them automatically when reconciliation shows an integer-ratio quantity change.

### 8.3 Money features
- Accounts list with computed balances; transfers between accounts (two linked rows).
- Bank-statement CSV import: upload → column mapping UI (date, description, debit, credit / amount+sign, balance) → saved mapping profile per bank → dedupe by hash → auto-categorise via `category_rules` → review/edit → commit.
- Transactions: filters, bulk recategorise, "create rule from this".
- Budgets: per category per month, progress bars, carry-over option.
- Liabilities: amortisation schedule, outstanding principal as of today, upcoming EMIs.

### 8.4 Investments
- Holdings derived from `investment_txns` (FIFO lots). Values: equities/ETFs from EOD latest close; MFs from AMFI NAV; FD by accrual formula; PPF/EPF/NPS/real estate/other from latest `asset_valuations`.
- Per holding and portfolio: invested, current value, unrealised/realised P&L, day change, **XIRR** (Newton-Raphson with bisection fallback; unit-tested), holding period, LTCG/STCG split.
- Overview: allocation by asset class / sector (sector from fundamentals or a symbol→industry map) / market-cap bucket, concentration warnings (single stock > 10%, sector > 30%; thresholds in settings).
- Stock holding detail: price chart with buy/sell markers, lots table, technical hits, fundamentals link.

### 8.5 Trading journal
- Trades table + form (planned trades can be pre-filled from strategy pages; `setup` defaults to the strategy id).
- Per trade: P&L (₹, %), R-multiple (needs stop), holding days, MAE/MFE from EOD data.
- Analytics: equity curve, win rate, avg win/loss, profit factor, expectancy, max drawdown, streaks, P&L by setup / month / weekday, calendar heatmap, R-multiple distribution. Filters by date range, setup, symbol, side.

### 8.6 Home dashboard
KPI cards (net worth, month change, total investments, liabilities, this-month income/expense/savings rate, portfolio day change, portfolio XIRR, trading P&L MTD), net-worth history line (from snapshots), allocation donut, cash-flow bar chart (last 12 months), budget status, top movers in holdings, upcoming EMIs / insurance premiums / FD maturities (next 30 days), today's strategy hits on holdings + watchlist, "Last EOD update" status with the Update button.

### 8.7 Planning (phase 9)
Goals (required monthly SIP via FV of annuity at an assumed return; progress from linked instruments), capital-gains report for a selected FY from realised lots (equity STCG/LTCG with the applicable current rates and exemption stored in settings — do not hard-code rates in logic), insurance register with renewal reminders.

---

## 8A. Broker connectors (platform-agnostic, API-first)

Goal: fetch portfolio data **directly from broker APIs** (Zerodha Kite Connect first), with a design where adding another broker (Upstox, Groww, Angel One, Dhan, Fyers, ICICI Direct, …) is **one module folder + one config entry**, exactly like strategies.

### 8A.1 Principles
1. **Read-only.** FinBoard never places, modifies or cancels orders. Connector interfaces have no write methods, and each connector's HTTP client enforces an allow-list of endpoints (GET data endpoints + the session token exchange/logout only). A test asserts this.
2. **Canonical models only downstream.** Connectors translate broker payloads into the canonical types below; nothing outside a connector folder knows broker field names.
3. **API first, files as fallback.** A connector may declare file importers (e.g. Zerodha Console tradebook) for history the API cannot provide. API data and file data flow through the same canonical pipeline and dedupe against each other.
4. **Multiple connections per broker** (e.g. own + spouse's Zerodha accounts); every record is tagged with `connection_id`.
5. **Instrument identity by ISIN**, mapped to the NSE symbol used by the market store; broker trading symbols (e.g. `-BE` suffixes) are kept only as metadata.
6. **Secrets never leave the server**, are encrypted at rest, and are never logged.

### 8A.2 Contract (`src/core/broker/define.ts`)

```ts
// ---- canonical models ----
type Segment = "EQ" | "MF" | "FO" | "CDS" | "COM";
interface CanonicalTrade {
  externalId: string;            // broker trade id (unique per connection)
  orderId?: string;
  isin?: string; brokerSymbol: string; exchange: "NSE" | "BSE" | string; segment: Segment;
  side: "BUY" | "SELL"; quantity: number; price: number;
  tradedAt: string;              // ISO, IST
  product?: "CNC" | "MIS" | "NRML" | "MTF" | string;
  charges?: { brokerage?: number; stt?: number; exchange?: number; gst?: number; sebi?: number; stamp?: number };
  raw?: unknown;                 // stored in broker_raw_records, never used downstream
}
interface CanonicalHolding { isin: string; brokerSymbol: string; exchange: string; quantity: number; t1Quantity?: number; pledgedQuantity?: number; averagePrice: number; lastPrice?: number; asOf: string; }
interface CanonicalPosition { brokerSymbol: string; exchange: string; product: string; quantity: number; averagePrice: number; pnl?: number; asOf: string; }
interface CanonicalOrder { externalId: string; status: string; side: "BUY" | "SELL"; brokerSymbol: string; quantity: number; filledQuantity: number; price?: number; averagePrice?: number; placedAt: string; tag?: string; }
interface CanonicalFunds { available: number; used: number; total: number; asOf: string; }
interface CanonicalMfHolding { isin: string; schemeName: string; folio?: string; units: number; averageNav: number; lastNav?: number; asOf: string; }
interface CanonicalMfTxn { externalId: string; isin: string; side: "BUY" | "SELL"; units: number; nav: number; amount: number; date: string; }

// ---- connector ----
type AuthKind = "oauth-redirect" | "api-key-totp" | "api-key" | "file-only";

export function defineBroker<C extends z.ZodObject<any>>(b: {
  id: string;                          // "zerodha"
  name: string;                        // "Zerodha"
  credentials: C;                      // Zod schema for user-entered creds (e.g. apiKey, apiSecret) → rendered as a form
  auth: {
    kind: AuthKind;
    sessionTtl: "daily" | "long-lived" | { hours: number };      // drives "reconnect" banners
    begin?(ctx: AuthCtx<C>): Promise<{ redirectUrl: string }>;  // oauth-redirect
    complete?(ctx: AuthCtx<C>, query: URLSearchParams): Promise<Session>;
    login?(ctx: AuthCtx<C>, input: { totp?: string }): Promise<Session>; // api-key-totp
    logout?(ctx: AuthCtx<C>, s: Session): Promise<void>;
  };
  capabilities: {
    holdings?: true; positions?: true; funds?: true; orders?: true;
    trades?: { history: "today" | { maxDays: number } | "unlimited" };
    mfHoldings?: true; mfTransactions?: { history: "today" | { maxDays: number } | "unlimited" };
  };
  fetch: {                             // implement exactly the declared capabilities
    profile(s: Session): Promise<{ clientId: string; name?: string; email?: string }>;
    holdings?(s: Session): Promise<CanonicalHolding[]>;
    positions?(s: Session): Promise<CanonicalPosition[]>;
    funds?(s: Session): Promise<CanonicalFunds>;
    orders?(s: Session): Promise<CanonicalOrder[]>;
    trades?(s: Session, range: { from: string; to: string }): Promise<CanonicalTrade[]>;
    mfHoldings?(s: Session): Promise<CanonicalMfHolding[]>;
    mfTransactions?(s: Session, range: { from: string; to: string }): Promise<CanonicalMfTxn[]>;
  };
  importers?: Array<{                  // file fallbacks, same canonical output
    id: string; label: string; accept: string[];                 // e.g. [".csv", ".xlsx"]
    produces: "trades" | "holdings" | "mfTransactions" | "mfHoldings";
    detect(headers: string[]): boolean;                          // auto-detect file type on upload
    parse(file: Buffer, filename: string): Promise<CanonicalTrade[] | CanonicalHolding[] | CanonicalMfTxn[] | CanonicalMfHolding[]>;
  }>;
  rateLimit?: { perSecond: number };
  docsUrl?: string;                    // shown on the connection setup screen
}): BrokerConnector<C>;
```

A conformance test suite (`src/core/broker/conformance.ts`) runs against every connector found in the config using recorded fixtures: declared capabilities are implemented, outputs validate against canonical Zod schemas, `externalId`s are stable across repeated fetches, and the endpoint allow-list contains no order-placement routes.

### 8A.3 Wiring through config

```ts
// finboard.config.ts
brokers: [
  { id: "zerodha", connector: () => import("@/modules/brokers/zerodha") },
  // { id: "upstox", connector: () => import("@/modules/brokers/upstox") },
],
```
Settings → **Connections** lists available brokers from this array and the user's connections; Investments, the journal and the sync engine iterate over connections generically.

### 8A.4 Sync engine (`src/core/broker/sync.ts`) — broker-agnostic
For each connection with a valid session, and for each declared capability:
1. Fetch (respecting `rateLimit`, retry with backoff on 429/5xx; on auth errors mark the connection `expired` and stop).
2. Validate against canonical schemas; store raw payloads in `broker_raw_records` (upsert on `(connection_id, kind, external_id)`), for audit and re-processing.
3. Normalise: trades/MF txns → `investment_txns` (upsert on `(connection_id, external_id)`; also dedupe API vs file-import rows by `trade_id`, falling back to a hash of `isin + tradedAt + side + qty + price`); holdings/MF holdings → `broker_holdings_snapshots` (one per connection per day); funds → cash balance of the linked demat/trading `account`.
4. Re-run trade matching (journal) and reconciliation (§8.2) for affected symbols.
5. Record a `broker_sync_runs` row with counts and errors.
6. **Gap tracking**: maintain `broker_sync_coverage(connection_id, kind, date)`. If the connector's trade history is `"today"` and a trading day has no coverage, flag it in the UI ("Trades for 3 Nov were not synced — import a tradebook for that range"), and let the matching file importer fill it.

Triggers: "Sync now" (per connection or all), automatically right after a successful login callback, and as a `broker-sync` phase in the Update EOD flow (§5.3; skipped with a notice if the session is expired). CLI: `npm run broker:sync [-- --connection <id>]`.

### 8A.5 Zerodha connector (`src/modules/brokers/zerodha/`) — first implementation
**Verify every detail below against the current Kite Connect v3 docs (https://kite.trade/docs/connect/v3/) and the developer console before coding; the docs win if anything differs.**
- **App setup (owner does once)**: create an app at developers.kite.trade. A personal-use plan with no market-data access is enough, because FinBoard only needs portfolio/order data (prices come from the owner's EOD files). Register the redirect URL `http://127.0.0.1:3000/api/brokers/zerodha/callback` (v2: the Vercel HTTPS URL). The owner enters `apiKey` and `apiSecret` in Settings → Connections; they're stored encrypted (§8A.6).
- **Auth (`oauth-redirect`, `sessionTtl: "daily"`)**: `begin` → `https://kite.zerodha.com/connect/login?v=3&api_key=<apiKey>`; callback receives `request_token` (+ `status=success`); `complete` → `POST https://api.kite.trade/session/token` with `api_key`, `request_token`, `checksum = SHA-256(api_key + request_token + api_secret)` → `access_token`. Requests use headers `X-Kite-Version: 3` and `Authorization: token <api_key>:<access_token>`. The token expires daily (early morning, next day) and has no refresh token, so the UI shows a one-click **"Reconnect Zerodha"** banner when expired. Verify the `state`/redirect params to prevent CSRF (store a nonce in a short-lived signed cookie and pass it via the `redirect_params` option).
- **Endpoints used (allow-list)**: `GET /user/profile`, `GET /user/margins`, `GET /portfolio/holdings`, `GET /portfolio/positions`, `GET /orders`, `GET /trades`, `GET /mf/holdings`, `GET /mf/orders`, plus `POST /session/token` and `DELETE /session/token`. Use a thin typed `fetch` client (Zod-validated responses); the official `kiteconnect` npm SDK is acceptable only if wrapped so the allow-list is still enforced.
- **Capabilities**: holdings, positions, funds, orders, `trades: { history: "today" }`, mfHoldings, `mfTransactions: { history: { maxDays: <per docs> } }`. Note that `/orders` and `/trades` return **only the current trading day**, so daily syncs (via the Update EOD flow) build history going forward.
- **History bootstrap and gap filling via importers** (same connector):
  - `console-tradebook` — Zerodha Console tradebook CSV (Equity and MF segments). Expected headers: `symbol, isin, trade_date, exchange, segment, series, trade_type, auction, quantity, price, trade_id, order_id, order_execution_time` (header-name based, verify against fixture).
  - `console-holdings` — Console holdings CSV (used as a reconciliation snapshot).
  - `console-pnl` (optional) — tax P&L statement, used only to cross-check realised P&L.
- **Charges**: Kite APIs don't return per-trade charges; estimate them with a configurable charge schedule (brokerage, STT, exchange txn, SEBI, stamp duty, GST) stored in settings, flagged as estimates; contract-note import can be added later.
- **Rate limit**: stay well under Kite's documented per-second limits (`rateLimit: { perSecond: 3 }` default).

### 8A.6 Security
- Broker credentials and session tokens are stored in `broker_connections` encrypted with **AES-256-GCM** using `FINBOARD_ENCRYPTION_KEY` (32-byte, base64, from `.env.local`; startup fails with a clear message if missing while a connection exists). Rotating the key re-encrypts rows via `npm run secrets:rotate`.
- Tokens are never sent to the client, never logged (central logger redacts known secret fields), and are deleted on "Disconnect" (also calling the broker's logout endpoint if available).
- OAuth callbacks validate the CSRF nonce; callback routes reject requests when `AUTH_MODE=google` and the user isn't signed in.

### 8A.7 UI
- **Settings → Connections**: cards per connection (broker logo text, label, client id, status chip `connected | expired | error | disconnected`, session expiry time, last sync time + counts, coverage gaps), buttons Connect / Reconnect / Sync now / Disconnect / Import file. "Add connection" wizard: choose broker (from config) → enter credentials (form from the connector's Zod schema; shows the connector's setup notes + `docsUrl`, including the redirect URL to register) → authenticate → first sync → offer history bootstrap via file importer.
- Global top-bar indicator when any connection is expired, with one-click reconnect.
- Investments → Stocks / Mutual Funds: filter by connection; each holding shows "broker qty vs derived qty" badge when they differ.
- Sync log page: recent `broker_sync_runs` with expandable errors.

### 8A.8 Adding another broker (must be true when done; document in README)
1. Create `src/modules/brokers/<id>/index.ts` exporting `defineBroker({...})` (plus any helper files and fixtures in that folder).
2. Add `{ id: "<id>", connector: () => import("@/modules/brokers/<id>") }` to `brokers` in `finboard.config.ts`.
The conformance suite then runs against it automatically; the Connections wizard, sync engine, holdings, journal and reconciliation work without other changes.

---

## 9. UI / UX requirements

- Layout: collapsible sidebar generated from the config (icons, nested groups, active state, breadcrumbs), top bar with global symbol search (universe + fundamentals + holdings), Update EOD button with status, theme toggle.
- Indian formatting utilities in `src/lib/format.ts` (₹, lakh/crore compact formatting, `dd MMM yyyy` dates, IST). Green/red for gains/losses with colour-blind-safe alternative (also use ▲/▼).
- Every page has loading skeletons, empty states with the next action ("Upload a Screener file", "Connect your broker", "Set FINBOARD_EOD_DIR"), and error states with actionable messages.
- Responsive down to 375 px width; tables scroll horizontally inside their container only.
- Keyboard: `/` focuses search; `u` triggers update (with confirm).
- A visible disclaimer on Research pages: signals and valuations are analytical tools, not investment advice.

---

## 10. Configuration & environment

`.env.example` (commit it; never commit `.env.local`):
```
FINBOARD_MODE=local                 # local | cloud (v2)
FINBOARD_EOD_DIR=/path/to/your/eod  # owner's existing data (read-only to FinBoard)
FINBOARD_INDEX_DIR=                 # optional index data
FINBOARD_DATA_DIR=/path/to/finboard-data   # signals, fundamentals, db, cache
EOD_UPDATE_COMMAND="python update.py"
EOD_UPDATE_CWD=/path/to/your/scripts
AUTH_MODE=none                      # none | google
AUTH_SECRET=
AUTH_GOOGLE_ID=
AUTH_GOOGLE_SECRET=
ALLOWED_EMAIL=
DATABASE_URL=                       # v2 Neon; empty → PGlite at $FINBOARD_DATA_DIR/db
FINBOARD_ENCRYPTION_KEY=            # base64 32 bytes; encrypts broker credentials/tokens (generate: openssl rand -base64 32)
FINBOARD_PUBLIC_URL=http://127.0.0.1:3000   # used to build broker OAuth redirect URLs
```
Broker API keys/secrets are **not** env vars — they are entered per connection in Settings → Connections and stored encrypted, so multiple accounts and brokers work the same way.
Validate env with Zod at startup (`src/env.ts`) and show a friendly setup screen if required paths are missing or unreadable.

---

## 11. Testing & quality gates

- **Unit (Vitest)**: indicators (vs hand-computed fixtures), each strategy on synthetic series that should and should not trigger (≥ 2 positive and 2 negative cases each, including edge cases like insufficient lookback), look-ahead check (appending future bars must not change `scan` result on a truncated series), Screener parser (fixture), each valuation model (hand-worked example), XIRR, FIFO lot matching, bank import dedupe, config validation test (§4.2).
- **Broker connectors**: conformance suite (§8A.2) for every configured connector using recorded fixtures (HTTP mocked with `msw` or `undici` MockAgent — no live broker calls in tests); Zerodha checksum/session exchange; endpoint allow-list (no order routes); API-vs-file dedupe (same trade imported both ways yields one row); sync idempotency (running sync twice changes nothing); expired-token handling marks the connection `expired`; coverage-gap detection; encryption round-trip and that secrets never appear in logs or client payloads.
- **Parity test** for the owner's existing strategy: run the original script/logic and the port on the same fixture data; hit lists and key values must match exactly (document any intentional differences with the owner's approval).
- **E2E smoke (Playwright)**: app boots with fixture data dir, sidebar renders from config, a strategy page lists hits, a fundamental page shows a fair value, Update EOD runs a fake script (fixture command) and streams logs.
- **Extensibility test**: a test adds a dummy strategy node + module (in a temp config) and asserts it appears in the resolved nav, the technical dashboard card list, and the scan pipeline — without touching any other file.
- `npm run lint && npm run typecheck && npm test && npm run build` must pass at the end of every phase.

---

## 12. Phases (commit after each; each lists acceptance criteria)

1. **Foundation** — Next.js + TS strict + Tailwind + shadcn; `src/env.ts`; `finboard.config.ts` + `defineConfig` + resolver + catch-all route + sidebar/breadcrumbs from config; page registry with placeholder pages; PGlite + Drizzle schema + migrations; Auth.js wired behind `AUTH_MODE`; formatting utils. *Done when* every nav entry renders a placeholder and the config validation test passes.
2. **Discovery + market data** — complete `docs/DATA_CONTRACT.md` with the owner; `LocalFileMarketStore` with configurable column mapping and caching; index loading; `ScriptEodUpdater` + SSE job API + Update EOD button with live log; symbol page with candle chart; CLI scripts. *Done when* the owner's real data loads and the button runs their script end-to-end.
3. **Technical** — indicator library; `defineStrategy`; scan pipeline + RS rating + `SignalStore`; `StrategyPage` + chart drawer + presets; ABC, 44 MA, SEPA, VCP, Darvas, and the owner's strategy (with parity test); `TechnicalDashboard`; extensibility test. *Done when* a fresh scan over the full universe produces signals and every strategy page shows hits with overlays.
4. **Fundamental** — Screener upload + parser + derived metrics; `defineValuationModel`; all models; `ApproachPage` ×3; `FundamentalDashboard` with football-field chart. *Done when* uploading a fixture file yields fair values in all three approaches.
5. **Brokers + Investments** — `defineBroker` + canonical models + conformance suite; encryption; Connections UI and wizard; sync engine with coverage tracking; Zerodha connector (Kite Connect OAuth, holdings/positions/funds/orders/trades/MF) with Console file importers for history; `broker-sync` phase in Update EOD; reconciliation; FIFO holdings; XIRR; MF via AMFI; FD/PPF/EPF/NPS/other assets; Overview page. *Done when* the owner connects Zerodha, a sync populates holdings that reconcile with the broker snapshot, a one-time Console tradebook import back-fills history without duplicates, and a dummy second broker added via the two-step pattern passes the conformance suite.
6. **Trading journal** — trade matching, journal CRUD, plan-from-strategy flow, analytics.
7. **Money** — accounts, transactions, bank CSV mapping import, rules, budgets, liabilities + amortisation.
8. **Home dashboard** — net-worth snapshots (also written by the update flow) and all widgets.
9. **Planning** — goals, capital-gains report, insurance.
10. **Polish & docs** — empty/error states, responsiveness pass, seed/demo mode (`npm run seed` with synthetic data so the app can be shown without real data), `README.md` (local setup, data contract, how to add a strategy/valuation model in two steps with a worked example, v2 Vercel migration notes).

---

## 13. "How to add a strategy" (must be true when you finish; put this in the README)

1. Create `src/modules/technical/my-strategy.ts`:
   ```ts
   export default defineStrategy({
     id: "my-strategy", name: "My Strategy", description: "…rules…",
     params: z.object({ period: z.number().int().min(2).default(20) }),
     lookback: 60,
     scan(series, p) { /* return StrategyHit | null */ },
     overlays(series, p) { return []; },
   });
   ```
2. Add `{ id: "my-strategy", title: "My Strategy", strategy: () => import("@/modules/technical/my-strategy") }` under the Technical node in `finboard.config.ts`.

Result: new sidebar entry, page at `/research/technical/my-strategy`, card on the Technical dashboard, included in every scan. Same two-step pattern for valuation models (file in `src/modules/valuation/<approach>/` + entry in the approach's `models` array) and for brokers (folder in `src/modules/brokers/<id>/` + entry in `brokers`, see §8A.8).

---

## 14. Out of scope for v1

NSE data downloading/backfill (owner's script handles it), backtesting engine, live/intraday quotes, any order placement or trading through broker APIs (FinBoard is read-only), broker market-data/historical-candle APIs, broker postback webhooks (need a public URL; consider in v2), connectors other than Zerodha (the framework must support them), multi-user support, mobile app, notifications by email/push, cloud deployment. Keep the seams described in §5.4 so these can be added later.
