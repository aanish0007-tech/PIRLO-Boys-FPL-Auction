# Starter prompt for Claude / ChatGPT

Attach BOTH files (`fpl-auction.html`, `PROJECT-CONTEXT.md`) to the conversation, then paste this:

---

You are taking over an existing, working project: a single-file web app called "Conran Smith Lane" for running a live in-room Fantasy Premier League (FPL) auction for the 2026/27 season with 9–10 friends. I've attached:

1. `fpl-auction.html` — the ENTIRE app (vanilla HTML/CSS/JS, no build step, no framework). This is the current production version, already deployed on Netlify and working.
2. `PROJECT-CONTEXT.md` — full documentation: product rules, state schema, architecture, API/CORS strategy, Excel import/export format, UI conventions, and a list of INVARIANTS that must be preserved.

Before making any change:
- Read PROJECT-CONTEXT.md fully, especially the "Invariants to preserve" section. These are non-negotiable product rules (sealed random auction order, the £6.0m pool guarantee, the max-bid reserve formula, budget consistency across all mutation paths, single-file architecture).
- Note that state lives in localStorage under the key `rostrum-fpl-2627` and any state-shape change requires updating the `migrate()` function so existing saved auctions and old Excel/JSON exports still load.

Working rules:
- Keep everything in the single HTML file. CDN `<script>` tags are allowed (SheetJS is already used for Excel).
- When you change the JS, mentally trace every mutation path (hammer, log edit ✎, log revert ↩, pass-undo, manual squad add/edit/delete, Excel import) to confirm budgets and ownership stay consistent.
- Return the COMPLETE updated `fpl-auction.html` after each change — not a diff or snippet — since I deploy by dragging the whole file into Netlify. If a change affects deployment routing, also return an updated `netlify.toml` (its current content is documented in PROJECT-CONTEXT.md).
- After deploying I always hard-refresh; remind me if a change also needs a state reset or a pool re-fetch.

My first request is: [DESCRIBE YOUR CHANGE HERE]

---

## Tips for the collaborator

- Also keep `netlify.toml` from the deploy folder; it must accompany the HTML in every Netlify deploy or the FPL API proxy breaks.
- To test locally: double-click the HTML. The FPL fetch may fall back to public CORS proxies locally; the deployed Netlify site is the reliable path.
- Before experimenting on real auction data, click **Export .xlsx** in the app header — that workbook restores the exact state via **Import .xlsx**.
