# Conran Smith Lane — FPL Auction App: Project Context

A single-file web app for running a live, in-room Fantasy Premier League auction for the 2026/27 season with 9–10 friends, plus a live leaderboard once the season starts. Built as one self-contained `fpl-auction.html` (vanilla HTML/CSS/JS, no build step) deployed on Netlify alongside a `netlify.toml`.

## Files

| File | Purpose |
|---|---|
| `fpl-auction.html` | The entire app: markup, CSS, and JS in one file. |
| `netlify.toml` | Netlify redirect rules. Proxies `/fpl-api/*` → `https://fantasy.premierleague.com/api/*` server-side (bypasses FPL's CORS block) and serves the app at the site root. **Must be deployed together with the HTML** — Netlify drag-and-drop deploys replace the whole site each time. |

## How the auction works (product rules — preserve these)

1. **Setup** (tab 1): fetch the player pool live from the FPL API, name the managers, set budget (£100 default), timer, opening bid (£1), min raise (£1), squad quotas (2 GK / 5 DEF / 5 MID / 3 FWD = 15 players each).
2. **Pool builder**: targets ~200 players. **Hard guarantee: every player priced ≥ the "Always include" threshold (default £6.0m) is always in the pool** — no potential Tier 1/2 player may ever be missing. Remaining room up to the target is filled with the best cheaper players, allocated by position need (quota × managers − guaranteed count). If guaranteed players alone exceed the target, the pool runs over by design.
3. **Tiers**: exactly 3. Defaults: T1 ≥ £7.5m, T2 ≥ £5.5m, T3 below. Configurable in setup.
4. **Fair sealed queue** (anti-gaming — the core feature): the auction order is a *stratified jittered shuffle* — each tier's players get evenly spread random positions across the whole auction, so premium players keep appearing until the very last lots. The upcoming order is **never displayed** (only per-tier remaining counts). This defeats the "wait everyone out, then cherry-pick" strategy. Uses `crypto.getRandomValues`.
5. **Bidding happens verbally in the room.** The app only records results: tap the winning manager → set final price → "Sold". Bids open at £1 regardless of FPL list price. The countdown timer is a pacing tool only — at 0 it prompts "record the result", it never auto-assigns.
6. **Max bid rule**: `balance − (unfilled slots − 1) × openingBid` — every manager must keep £1 for each *other* unfilled slot, so nobody can hoard and the final slot can use the full remaining balance. Enforced in the auction, log edits, and manual squad assignment.
7. **Passed lots** recycle to a random later queue position (can be undone from the log). Any sale can be edited (✎ winner/price) or reverted (↩) from the Hammer log at any time.
8. **Post-auction manual assignment**: after the pool runs out, remaining slots are filled by preference — empty cells on the Squads board have "＋ add" with autocomplete (unowned players of that position only). Validations: no double ownership, position must match slot, price ≥ £1 and ≤ max-bid cap.
9. **Leaderboard**: no FPL league is used. On demand, the app fetches `bootstrap-static` and sums each manager's owned players' `event_points` (GW) and `total_points` (season). Shows last-updated timestamp. Pre-GW1, totals are FPL's 2025/26 carryover (labelled).

## Architecture

- **State**: single object `S`, persisted to `localStorage` under key `rostrum-fpl-2627`, saved via `save()` after every mutation. `migrate()` upgrades old saved shapes (runs on load and JSON import).
- **State shape** (abridged):
  ```js
  S = {
    players: [{ id:"fpl123", name:"Bukayo Saka" /*full*/, web:"Saka" /*FPL short*/,
                team:"ARS", pos:"GK|DEF|MID|FWD", price:9.5 /*FPL list £m*/,
                tier:1|2|3, pts, ppg /*last-season stats*/,
                status:"available"|"sold", ownerId:null|number, paid:number }],
    participants: [{ id:number, name, budget, spent }],
    settings: { budget, timer, openBid, minBid, quota:{GK,DEF,MID,FWD},
                minPrice, tiers:[t1,t2], targetPool, guaranteeMin, sound },
    queue: ["fpl123", ...],   // sealed auction order (player ids)
    qIndex: number,           // current position in queue
    onBlock: { pid, bid, bidderId } | null,
    log: [{ type:"sold"|"passed", playerId, name, buyerId, buyer, amount, ts }],
    board: { ts, gw, rows } | null,   // cached leaderboard
    started, view, theme, statsLabel ("2025/26"|"2026/27"), loadedAt, rawLoaded
  }
  ```
- **Single source of truth**: squads, pool, remaining list, log, and leaderboard are all derived from `players[].ownerId/paid` + `participants[].budget/spent`. Any mutation path (hammer, log edit/revert, manual squad assign) must keep those consistent — budgets are always recomputed pairwise (refund old, charge new).
- **Views/tabs**: `setup`, `auction`, `players`, `remaining`, `squads`, `board`. Hash-routed (`#squads` etc.) so tabs can be opened in separate windows. `switchView()` → `renderView()` dispatch. A `storage` event listener syncs other open windows of the same browser automatically. **State does not sync across different devices** — run auction night from one machine.
- **Key functions**: `buildQueue()` (stratified shuffle), `nextLot()`, `hammer()`, `selectWinner()/setPrice()`, `maxBid()/eligible()/slotsLeft()/quotaRoom()`, `trimPool()` (pool builder), `manualAssign()` (squad cell editor), `revertSale()/saveEdit()/undoPass()/undoLast()`, `loadFromFPL(auto)`, `fetchScores()/renderBoard()`, `exportXLSX()/importXLSX()`.

## FPL API & CORS (important)

The FPL API (`https://fantasy.premierleague.com/api/...`) **blocks all cross-site browser requests** (CORS). `fetchJSON()` tries routes in order:
1. `/fpl-api/*` — same-origin Netlify proxy (the reliable path; requires `netlify.toml`). Skipped on `file://`.
2. Direct fetch.
3. Public CORS proxies (corsproxy.io, allorigins raw/get, codetabs) — flaky fallbacks for local use.

All fetch URLs carry a `?_=Date.now()` cache-buster. Endpoints used: `bootstrap-static/` only (players, teams, prices, points, events). `elements[].now_cost` is in tenths of £m (60 → £6.0). Positions: `element_type` 1–4 → GK/DEF/MID/FWD.

**Real-time behaviour**: the pool auto-re-syncs from the API on app open (throttled to 5 min) *only while the auction hasn't started*. Once started, the pool is frozen for fairness. The leaderboard always fetches live.

## Excel export/import

Header buttons produce/consume a 5-sheet workbook via SheetJS (CDN: cdnjs xlsx 0.18.5): **Players** (ID, Name, WebName, Club, Position, ListPrice, Tier, SeasonPts, PPG, Status, OwnerID, Owner, Paid), **Managers**, **Log**, **Queue** (sealed order), **Settings** (key/value; quota & tiers as JSON strings). On import, the **Players sheet is the source of truth**: status derives from OwnerID, and every manager's spent/balance is recomputed from Paid — so hand-edits in Excel reconcile cleanly. Sold players missing log rows get synthetic ones so ✎/↩ keep working. Legacy JSON imports still accepted.

## UI conventions

- Full player names in Auction / Player Pool / Remaining / Hammer log; **short names** (`web` field, fallback last word) on the Squads board and Leaderboard to keep columns narrow. Full name in tooltips.
- Squads tab is a full-width (`.view.wide`, up to 97vw/1840px) excel-style grid: `table-layout:fixed`, equal columns, ellipsis truncation — 10 managers fit without horizontal scroll on ≥~1150px screens; below that it scrolls (min-width 1080px).
- Light/dark themes via `data-theme` on `<html>` + CSS variables. Fonts: Chakra Petch (display/numbers) + Instrument Sans (body), Google Fonts CDN.
- Toasts for all feedback; confirm() before destructive actions; keyboard shortcuts on the auction screen (1–9 select winner, ↑/↓ price, Enter sold, Space pause).

## Invariants to preserve when making changes

1. Never display the upcoming queue order.
2. Never drop a player priced ≥ `guaranteeMin` from the pool.
3. `sum(owned paid) === spent` and `budget === settings.budget − spent` for every manager, after every mutation path.
4. A player has at most one owner; ownership changes always refund/charge correctly.
5. Max-bid reserve rule applies to every acquisition path.
6. Pool frozen after `started === true`; only ownership/paid change after that.
7. `migrate()` must keep old localStorage/exports loadable when you change the state shape.
8. Keep it a single HTML file (plus netlify.toml); CDN scripts are fine.

## Known limitations / candidate improvements

- No cross-device sync (localStorage only). A backend or Firebase/Supabase layer would enable true multi-device viewing.
- Leaderboard counts all 15 owned players (no best-XI/captain logic).
- CSV-loaded pools can't map to FPL ids, so live scoring requires an API-loaded pool.
- Public CORS proxies for local (non-Netlify) use are unreliable by nature.
- No automated tests; JS is syntax-checked with `node --check` during development.

## Deployment

Put `fpl-auction.html` + `netlify.toml` in one folder → drag into Netlify (app.netlify.com/drop or the site's Deploys page). Both files every time. Hard-refresh (Ctrl/Cmd+Shift+R) after deploying — browsers cache single-file apps aggressively. Also runs locally by double-clicking the HTML (API then depends on public proxies; CSV upload is the offline fallback).
