# Yalvon360 V24 — Ready to deploy on Render (real bug fixed + verified)

## Deploying to Render

1. **Push this folder to a GitHub repo** (create one if you don't have
   it yet — Render deploys from GitHub, not from a zip upload).
2. **Sign up at [render.com](https://render.com)** and connect your
   GitHub account.
3. **New → Web Service**, pick the repo you just pushed.
4. Render should auto-detect Python. Set these if it doesn't pick them
   up from `render.yaml` automatically:
   - **Build Command**: `pip install -r requirements.txt`
   - **Start Command**: `gunicorn --workers 1 --threads 4 --timeout 120 app:app`
   - **Plan**: Free (to start)
5. Click **Create Web Service**. First build takes a few minutes; you'll
   get a live URL like `yalvon360.onrender.com`.
6. *(Optional)* Add `MARKET_DATA_API_KEY` under the service's
   **Environment** tab if you have a Twelve Data key, for live world
   indices/forex/commodities instead of the built-in dev values.

**Why `--workers 1`, specifically:** this app runs one background thread
(`start_bulk_refresh_thread` in `app.py`) that refreshes live PSX/crypto/
forex/fund prices on a timer. Gunicorn's normal multi-worker model spawns
a separate OS process per worker — each one would run its own independent
copy of that thread, meaning every external site gets hit N times instead
of once per cycle. `--threads 4` handles concurrent visitors within that
one worker instead, which is enough for personal or small-team traffic.

**Real bug fixed and verified before this was written:** `init_db()` and
`start_bulk_refresh_thread()` were previously called only inside
`if __name__ == "__main__":` — which never runs under a production WSGI
server like gunicorn (it imports `app` as a plain module and calls the
`app` object directly). Deployed as-is, the site would have "gone live"
but every database-touching request would have failed, and prices would
never have refreshed. Fixed by moving both calls to run unconditionally
on import. **Verified, not assumed**: started a `wsgiref`-based server
that imports `app.py` exactly the way gunicorn does (never touching
`__main__`), and confirmed `portfolio.db` gets created and the background
thread actually runs — `/api/stocks/live` returned real data purely from
that import path.

### The one real limitation worth knowing before you deploy

Render's **free tier has no persistent disk** — `portfolio.db` resets to
a fresh, empty database every time the service restarts, redeploys, or
wakes up from sleep (free services sleep after ~15 minutes of no
traffic). Your portfolio/watchlist/recorded history will not survive
that. Three ways to handle it:
1. **Accept it for now** — fine for testing/sharing a demo link.
2. **Upgrade to a paid Render plan** and attach the persistent disk
   defined in `render.yaml` (commented out by default since it only
   works on paid plans).
3. **Move to a Postgres database** instead of SQLite (Render's free
   Postgres trial persists independently of the web service) — a
   bigger change than this zip includes, ask if you want it built.

### Domain

Render gives you a free `*.onrender.com` subdomain immediately. To use
your own domain: buy one (Cloudflare Registrar or Namecheap are honest
picks — check renewal price, not just the first-year promo), then add it
under the Render service's **Settings → Custom Domains** and point your
registrar's DNS at Render per their instructions. Render's infrastructure
is IPv4-only — don't add AAAA (IPv6) DNS records, they'll cause
intermittent failures for IPv6 users.

---

This round was a genuine "take the time" pass — went back through V22's
work looking for gaps between "the route exists" and "you can actually
click something," and did real, itemized testing (comparing exact JSON
shapes returned by each route against what the frontend expects, not just
checking for HTTP 200) rather than adding more surface-level features.

## Real bug found: Macro page never got the 18-indicator expansion

V22 expanded Pakistan macro data to 18 indicators, but that expansion only
reached the "Pakistan Profile" mini-sections on Markets/Dashboard — the
actual standalone **Macro page** was still stuck on the old 4-item list,
because it read from a different, smaller dataset. Fixed: the Macro page
now shows all 18 indicators, grouped into 7 real categories (Monetary,
External, Trade, Energy, Growth, Prices, Currency), each clickable.

## Wired up three click-throughs that had working routes but no way to open them

- **World/index market cards** (Nikkei, FTSE, Dow, Nasdaq, DAX, Hang Seng,
  Nifty, KSE-100, S&P 500, Tadawul, Bitcoin, Gold, WTI, USD strength — 15
  total) now open a real detail modal (52-week range, description,
  live/dev source).
- **Every Pakistan Macro indicator card** now opens a detail modal
  (value, trend, category, note).
- **"N Funds" counts** under Top Stocks Three Ways now open a modal
  listing the actual fund names (from your real 392-fund MUFAP directory).

All three share one modal component (`marketDetailModal`) — verified by
fetching each underlying route directly and confirming the JSON field
names match exactly what the modal's rendering code reads (`m.range52w`,
`m.category`, `d.funds[].amc`, etc.), not just that they return 200.

## New: universal Watchlist search (item #9)

Replaced the old "type one PSX symbol" input with a real cross-asset
search — type "U" and see matching stocks, crypto, forex pairs, and mutual
funds together, each tagged with its type, live price, and an Add button.
Optional filter chips (All/Stocks/Funds/Crypto/Forex). New
`/api/search-all?q=` backend route, debounced client-side.

**Two real bugs caught and fixed while testing this, not before shipping:**
1. Stock search silently returned zero results whenever the live PSX
   symbol fetch failed (e.g., no internet) — it never fell back to the
   same `FALLBACK_QUOTES` data the rest of the app already uses
   consistently. Fixed, and verified "FFC" now correctly returns a real
   match with price/change data.
2. Forex search was structurally broken: it matched every single currency
   pair against every query, because the matching logic checked whether
   the query appeared anywhere in the string `"USD" + code` — and "USD"
   itself contains U, S, and D, so any of those letters matched
   everything. Fixed to match only the actual currency code. Verified:
   searching "U" now correctly returns just EUR and AUD instead of all
   12 pairs.

### Verified in this sandbox

Full endpoint sweep (17 routes, all 200), portfolio history period
filtering, screener preset execution, watchlist add/remove round-trip,
and a specific before/after test of both search bugs with real query
results compared, not just status codes. Python, JS, CSS all
syntax-validated; every element ID (including dynamically-suffixed ones)
cross-checked against the HTML.

## New this round, all verified live

**#14 — Expand + click-through world indices.** `MULTI_MARKET` grew from
8 to 15 (added Dow, Nasdaq, FTSE, DAX, Nikkei, Hang Seng, Nifty). New
`/api/market-detail/<key>` route returns 52-week range + description.
Verified: `curl /api/market-detail/nikkei` returns correct range and note.

**#22 — Clickable fund-holder counts.** New `/api/fund-holders/<symbol>`
route. Verified: `OGDC` (55 funds) returns 55 real fund names/AMCs drawn
from your actual 392-fund MUFAP directory — honestly labeled as
illustrative (exact per-stock holdings need a licensed feed).

**#11 — Pakistan Macro expanded 8 → 18 indicators.** Added KIBOR (1M/6M),
T-Bill yields (3M/12M), remittances, exports, imports, trade balance, GDP
growth, gold. Each now categorized (Energy/Prices/Monetary/External/
Trade/Growth). New `/api/macro-detail/<key>` route for click-through —
verified live for "petrol".

**#16 — Stocks added to Cross-Asset Highlights** (KSE-100 + S&P 500,
alongside the existing crypto/forex/commodities cards).

**#17 — Trending Stocks: real period recalculation.** Daily through 1
Year now genuinely rescale and re-rank (not just relabel) via
`/api/market360?trending_period=`. Verified live: the same stock (TICL)
shows +6.22% for Daily vs +276.95% for 1 Year — a real, different number,
not a cosmetic label swap.

**#8 + #20 — Interactive dual-line Portfolio chart with real hover
values.** Built a new chart engine (`renderInteractiveDualChart`) showing
Portfolio Value vs. Amount Invested (or Daily vs. Cumulative P/L, via a
toggle), with genuine hover tooltips showing exact date/value/invested/
P/L per point — not just a static line. Period buttons (Daily through 1
Year) backed by a new `/api/portfolio/history?period=` endpoint that
actually filters by date. Wired into both Dashboard and Portfolio pages.
Verified: `/api/portfolio/history?period=1M` returns correctly filtered
data.

**#18 — 52-Week Highs/Lows title clarified** to "Stocks Hitting a
52-Week High or Low Today" with an explicit high-vs-low explanation
(both sections already existed side-by-side; only the confusing title
needed fixing).

**#21 — Fear & Greed labeling.** Added an explicit "Development/demo
score" note next to the PSX sentiment score, so it's clear this is the
fallback value pending a live calculation feed — the hard-coded score
itself was never removed, per your instruction.

**#29 (partial) — Screener presets + "activate all options."** Added 12
one-click presets (Oversold, Overbought, Golden Cross, Death Cross, MACD
Bullish, Above 200-day Average, High Volume, Value P/E<10, Big
Gainers/Losers Today, Near 52-Week High/Low) that actually run against
real filter logic. Also wired the "LIVE" checklist checkboxes (previously
decorative) into real filtering for every item with a clean boolean
mapping (EMA20→above 20-day, SMA50/200, Golden/Death Cross, MACD, Volume
Confirmation). Verified live: the "Oversold" preset correctly returned 0
matches in this fresh sandbox (accurate — RSI needs accumulated price
history, which starts from zero on a new install; not a bug).

**#10 — Dashboard Watchlist mirror.** New scrollable box on Dashboard
showing your watchlist in the same order as the main Watchlist page,
auto-updating whenever you add/remove/reorder from either place (both
read from the same `/api/watchlist` data). Verified: added FFC via API,
confirmed it appears in the watchlist response Dashboard reads from.

### Verified in this sandbox (this round)

Ran the full server and confirmed: all 12 core endpoints return 200,
period-filtered portfolio history works, trending-stocks period
recalculation produces genuinely different numbers per period,
market-detail/macro-detail/fund-holders routes all return correct data,
screener presets execute real queries, watchlist add/mirror works, and
zero "Portfolio360" text anywhere in the served page. Python, JS, and CSS
all syntax-validated; every element ID (including dynamically-suffixed
ones) cross-checked against the HTML.

## Fixed: "What's Moving The Stock Markets" was showing nothing — real bug

Root cause: the movers (Top Gainers/Losers/Most Active) and the KSE-100
mini-index were populated by `loadMarket()`, a function I never extended
when I merged the Markets-page content into the Dashboard a few rounds
ago — it only ever wrote to the Markets page's plain element IDs, never
the Dashboard's `...Home`-suffixed copies, and it was never called at all
when navigating to Dashboard specifically. Fixed: `loadMarket()` now
renders into both contexts, and fires on Dashboard navigation too.
Verified via `curl` that `/api/market` returns real stock data and that
the fix is live in the served page.

## Item 4 — removed all visible "Portfolio360" wording

Grepped the entire served HTML for "portfolio360" (case-insensitive)
after the fix — zero matches. Internal code comments (not user-visible)
still reference it for maintainer context; nothing in the UI does.

## Item 6 — PSX Fear & Greed vs. Global sentiment, now correctly separated

- Renamed the page to "PSX Fear & Greed" and rewrote the subtitle to say
  exactly that, instead of the vague "composite market sentiment" wording
- Added a genuine new **Global & Composite Sentiment** section on that
  same page — 11 markets (PSX, US, Crypto, Forex, Commodities, UK, Europe,
  Japan, China/HK, India, Saudi Arabia), each with its own score and a
  click-through (crypto/forex/commodities route to their real pages; the
  rest route to Markets for now — see deferred list)
- Added a 6th component ("52-Week Highs/Lows") to the PSX score breakdown,
  alongside the existing Breadth/Momentum/Volume/Volatility/Macro

## Item 7 — portfolio summary boxes grouped

Current Value / Total Invested / Total P/L now sit inside one visually
distinct "Your Portfolio" container with its own header and a link to the
full Portfolio page, instead of floating as disconnected cards. PSX
Directory (which isn't a portfolio number) moved to its own small card
below.

## Item 1 (partial) — logo enlarged

Header logo height increased 44px → 64px, max-width cap raised so it
actually renders bigger instead of being clipped.

## Item 5 (partial) — graph lines strengthened

Increased the fill-area opacity under all charts (0.08 → 0.16, up to 0.18
on large charts) and thickened large-chart line strokes (2.2px → 3.2px)
so `.big-chart`/`.detail-chart` graphs read as bold, filled area charts;
small sparklines got a smaller bump (2px → 2.2px) to stay "simple" per
your later clarification, without hover tooltips.

### Verified in this sandbox

Ran the full server: all 12 core endpoints return 200, confirmed
`global_sentiment` (11 entries) and the 6-component `fear_greed` breakdown
via `/api/extras`, confirmed real stock data flows through `/api/market`,
and confirmed zero "Portfolio360" text in the served page. Python, JS, CSS
all syntax-validated; every dynamic/suffixed element ID cross-checked
against the HTML.

---

## Full 29-item status

**Done and verified live (across V21 + V22):**
- #4 Remove visible "Portfolio360" wording — done
- #6 PSX vs. Global sentiment separation + more PSX detail — done
- #7 Group 3 portfolio boxes into one container — done
- #8 Portfolio Trend: investment-value line + Daily/1W/2W/3W/1M/3M/6M/1Y
  period controls, real hover values — done
- #10 Dashboard Watchlist mirrors main Watchlist, scrollable — done
- #11 Pakistan Macro expanded to 18 indicators — done
- #14 World indices expanded (8→15) + click-through detail route — done
  (frontend click handlers to open the detail still need wiring — data
  and route are live and tested, the modal/panel UI isn't built yet)
- #15 Fix "What's Moving" showing nothing — done (real bug, not cosmetic)
- #16 Stocks added to Cross-Asset Highlights — done
- #17 Trending Stocks period selector with real recalculation — done
- #18 52-Week Highs/Lows title clarity — done
- #20 Hover/touch exact values — done for the Portfolio Trend chart;
  other large charts (stock detail, macro trend lines) not yet upgraded
- #21 Fear & Greed live-vs-fallback labeling — done (hard-coded score
  kept as instructed; a real calculated version is still open)
- #22 Clickable fund-holder counts — done (route + real data; frontend
  click handler to open it still needs wiring, same as #14)
- #29 (partial) Screener: 12 one-click presets + all "LIVE" checklist
  items now actually filter — done; provider-ready architecture for
  crypto/forex/commodity screening is still open
- #1 (partial) Enlarge logo — done; tickers sit above the nav bar rather
  than literally in the same row as the logo
- #5 (partial) Darker/stronger graph lines — done for fill+stroke weight
  and the new dual-line chart's explicit colors; other charts still
  default to brand-gold

**Not done — still substantial, scoped work each:**
- #2 Three market timelines replacing hero space
- #3 Composite sentiment expansion on the Dashboard mini-card (the full
  Global Sentiment section from #6 covers the Fear & Greed page; the
  small Dashboard card is still PSX-only)
- #9 Universal Watchlist search across all asset classes
- #12 Back-navigation returning to exact scroll position
- #13 Richer Forex/Crypto/Funds/Commodities graphs + detail pages
- #14/#22 Frontend click-through UI for the new market-detail and
  fund-holders routes (data/routes done, modal/panel not built)
- #19 Per-investor-category flow graphs with period filter
- #20 Hover values on remaining large charts (only Portfolio Trend done)
- #21 Real calculated Fear & Greed (fallback + labeling done, live calc
  not built)
- #23 Levels To Play: more stocks, light status colors, detail pages
- #24 Seasonality: full historical stats and per-stock drill-down
- #25 Clickable Pakistan Profile boxes (route done via #11, frontend
  click-through not wired)
- #26 Clickable "Everything Happening" cards
- #27 Global (not Pakistan-only) events, by region
- #28 Site-wide stronger colors (partial) + enforced Dashboard/main-page
  parity as an ongoing rule
- #29 Provider-ready screener architecture for non-PSX asset classes

Given the size, I'd still recommend picking your top 3-5 from what's left
so each gets a properly tested implementation rather than a rushed one.

---

## Fixed: browser confirm()/alert() replaced with in-site modal

Every `confirm()`/`alert()` call (portfolio delete, refresh actions, etc.)
now uses a proper in-page modal (`siteConfirm()` / `siteAlert()` in
`app.js`) matching the site's own design, with a clear Yes/No choice —
no more browser-native popups sliding down from the top of the tab.

## New: global ticker bar + header search

- The four live ticker strips (PSX/Crypto/Forex/Mutual Funds) moved from
  inside the Dashboard page to a global bar above the navigation — visible
  on every page now, "in front of the logo" as requested.
- New search button in the header opens a live type-ahead search box that
  filters PSX symbols/companies as you type and jumps straight to a
  stock's detail page on selection.
- The "N PSX symbols" pill in the header is now a clickable button that
  opens the full stock directory.
- Removed the large intro paragraph from the hero; a short tagline now
  sits next to the logo instead.

## Fixed: sectors were never actually only showing "5 of many" — there
## were only 5 sectors defined, period

Root cause: `SECTOR_PERFORMANCE` in `app.py` genuinely only had 5 sectors.
"View All" was already showing all of them — there just weren't more to
show. Fixed by expanding to 30 real PSX sector categories. Also added:
- **Sectors are now clickable** — click any sector row (Dashboard strip or
  full Sectors page) to jump to All PSX Stocks pre-filtered to that sector
- **Sort dropdown** on the full Sectors page (biggest gainers/losers/name)
- **Dismiss (×) button** on the Dashboard's condensed sector strip only —
  hidden sectors persist via `localStorage` and can be reset with one click

## Clarifying the "Top Holdings keeps reappearing" report

Top Holdings on the Dashboard mirrors your actual portfolio (sorted by
value) — there's no separate "remove from this widget" action because
removing something there would have to mean removing the holding itself
(which the × button on the Portfolio page already does, with the new
confirm modal). This wasn't a bug re-adding things behind your back; it's
by design, though I recognize the UI didn't make that obvious. Worth
revisiting if you want a "pin/hide from Top Holdings only" option distinct
from removing the position entirely.

### Verified in this sandbox

- Confirmed via API that sector count is now 30 (was 5)
- Confirmed the global ticker bar, header search, confirm modal, and
  sector sort dropdown are all present in the served HTML
- All 11 core endpoints return 200
- Python, JS, and CSS all syntax-validated clean

---

## What's deferred (given the scope of the original list)

This request included roughly 50 distinct items. The ones above got full,
tested implementations. Still open, roughly in the order they'd likely
matter most:

1. **Add Transaction**: asset-type-first flow (stock/crypto/forex/commodity
   → then search within that type) — portfolio backend currently only
   supports PSX stocks structurally; crypto/forex/commodity portfolio
   tracking would need real schema changes, not just UI
2. **Watchlist**: type-ahead search showing stock/crypto/forex/fund matches
   together with type labels as you type
3. **Portfolio**: sparkline graphs on the P/L summary cards themselves (not
   just per-holding), daily-vs-cumulative P/L toggle with a two-line hover
   chart, best/worst performer stat
4. **Screener**: one-click predefined filter presets, and a results-replace-
   filters-with-back-button flow
5. **All PSX Stocks**: remove pagination (show all on one page)
6. **Crypto/Forex/Commodities pages**: sortable columns, timeframe selector
   (24h/7d/30d), alphabetical sort, more visible chart lines
7. **Mutual Funds**: expandable rows / click-through detail view
8. **Market Snapshot, Macro, Fear & Greed**: click-through detail on each
   box, section-wise breakdown per market
9. **World Clock**: full redesign with a clock face per market + one
   combined large clock
10. **News**: multi-outlet fetching, organized by market
11. **Journal**: fix the selection bug, more articles, market-wise sections
12. **Tools**: calculation history per calculator, more color/detail
13. **Footer** with contact information
14. **Per-stock/asset alerts** (price alerts) across PSX/forex/crypto/
    commodities

Given the size, I'd suggest picking the 3-5 that matter most to you next
so each gets the same tested treatment as what's above, rather than
rushing all of them at once.

---

## Dashboard now includes everything from the Markets tab

Per request: took the entire long Portfolio360-style page that lived on
the separate Markets tab and merged it directly into Dashboard, below
everything already there (stat cards, portfolio charts, holdings, ticker
strips, sector/macro strips, Today Across All Markets, sentiment-by-market).
Dashboard is intentionally long now — a "Complete Market Detail" divider
marks where the merged section begins:

- Risk sentiment gauge + multi-market cards (again, in more detail)
- Cross-Asset Signals
- PSX movers (top gainers/losers/most active)
- Cross-Asset Highlights (crypto/forex/commodities)
- Trending Stocks
- Today's 52-Week Highs and Lows
- Who Is Buying Pakistan (fund flow matrix + chart)
- What Insiders Are Doing
- Market Sentiment history
- Non-Equity Sentiment and Breadth
- Top Stocks, Three Ways
- Levels To Play
- Seasonality In Play
- Pakistan Profile
- News / Announcements / Payouts
- Coming Up In Pakistan (calendar)

The Markets tab itself is unchanged and still has all of this too — both
pages now share the same data and the same rendering function.

### How it's built (worth knowing if you extend it)

HTML element IDs must be unique per page, so every duplicated element on
Dashboard got a `Home` suffix (e.g. `multiMarketCards` →
`multiMarketCardsHome`). Rather than duplicate ~300 lines of rendering
JavaScript, the whole detail-rendering block was extracted into one
function, `renderMarket360Detail(d, extras, suffix)`, called once with
`suffix=""` (Markets page) and once with `suffix="Home"` (Dashboard) from
a single shared `/api/market360` + `/api/extras` fetch.

### Bug found and fixed while testing this

The functions that populate world markets, the live ticker strips, and now
this new detail block were only ever wired to run on `go()` page
navigation — never on the initial page load. Since Dashboard is the
default landing page, this likely means "Today Across All Markets" (world
cards) and the ticker strips were sitting empty on first load in earlier
versions, only populating once you navigated away and back. Fixed by
adding `loadMarket360()`, `loadDashboardHighlights()`, and
`loadDashboardTickers()` to the app's startup sequence.

### Verified

- Cross-checked every suffixed ID (`XHome`) has both a plain and `Home`
  variant present in the HTML (scripted check, not just visual inspection)
- Confirmed via `curl` that the served `/` HTML actually contains the new
  `...Home` elements
- Ran all 11 core API endpoints against the final packaged copy — all
  return 200
- Python and JS both compile clean

---

## Fixed: hero was actually broken in the browser

A screenshot from the real rendered page showed the "atmospheric" hero
(added in V15, tweaked in V17) was genuinely broken: the body paragraph
text was invisible (leftover `rgba(255,255,255,.88)` white text color from
when the background was dark — never updated when the background became
light, so white-on-white/cream = invisible), and "Discover" / "Your
Portfolio" were not actually centered as intended, overlapping the
absolutely-positioned KSE-100 badge.

Rather than keep patching an increasingly fragile centered/absolute-
positioned/gradient-text-clip layout, it's **removed entirely**. The hero
is back to the simple, original, tested layout: a solid vivid gradient
card (cyan → pink → orange, no navy), left-aligned copy with white text
that's actually readable against the saturated gradient, and the KSE-100
stat as a simple flex sibling on the right — no absolute positioning, no
background-clip text tricks, no centering bugs possible.

Also removed: the now-unused Google Fonts import (Playfair Display /
Dancing Script) and ~120 lines of dead CSS for the removed hero variant.

### Verified

Rendered a pixel preview of the restored hero using the exact same CSS
values (gradient stops, button styles, badge layout) to visually confirm
correct alignment and contrast before packaging — included as
`static/hero_preview_reference.jpg` isn't shipped in this zip (it was a
verification step, not a site asset), but you can regenerate it by running
the app.


The dark navy "atmospheric" hero (added in V15 to mirror a reference
image) has been replaced with a light, bright version — white/cream
background, vivid diagonal cyan/pink/orange/green accent streaks, a
gradient-clipped navy-to-cyan-to-pink italic serif heading, and solid
bright-cyan/white CTA buttons instead of dark glass buttons. The rest of
the site was already light-themed (confirmed: `--bg: #FAFBFE`, white
cards, light `rgba(250,247,240,.86)` nav bar) — this was the one
remaining dark section.

A rendered preview (`light_hero_preview.jpg`, built with the same
palette/fonts as the real CSS) is included in this delivery so you can
see it before running the app.

---

## Fixed: "Today Across All Markets" only showed PSX data

The world-market data was actually there all along (`/api/market360` was
correctly returning S&P 500, Bitcoin, Gold, WTI, Tadawul, forex — verified
with a live curl test), but the two card grids (PSX indices vs world
markets) had no label distinguishing them, so they just blended into one
generic-looking row. Fixed by adding clear **"🇵🇰 PSX Indices"** and
**"🌍 World Markets"** sub-labels above each grid, so both groups are
unmistakably visible. Also cleaned up a redundant duplicate
`loadDashboardHighlights()` call found while investigating this.

## Beautified: header

Restructured into a clean two-row layout:
- **Row 1**: logo (left) and status/Add Transaction (right)
- **Row 2**: full navigation, organized into three visually-grouped
  clusters (Overview / Markets / Research), each with a light background
  panel and a visible uppercase micro-label — previously these group
  labels were hidden on desktop
- **Active tab**: now a filled brand-gradient pill with a subtle shadow,
  instead of a flat tint, for a more premium feel
- **Card hover effect**: strengthened — cards now lift further, gain a
  stronger shadow, and pick up a cyan border on hover (index cards, world
  market cards, commodity cards)

---

## Fixed: graphs missing from cards (the repeated request)

Sparklines existed on Portfolio holdings and Mutual Funds, but never made
it onto the Dashboard/Markets index, world-market, and commodity cards
despite being asked for repeatedly. Fixed:

- **Real 7-day crypto sparklines** — turned on CoinGecko's free
  `sparkline_in_7d` parameter (`sparkline: "true"` in `fetch_crypto_live`,
  downsampled via `_downsample()` to ~24 points before sending to the
  browser). Applied to the Crypto top-6 strip and a new "7d Trend" column
  in the main crypto table, tagged "7d" so it's clearly real data.
- **Every dev-labeled card now has a trend line** — PSX index cards, world
  market cards (Dashboard + Markets page), and commodity cards all show a
  small graph via a new `seededSparkline()` in `app.js`. Since the
  underlying numbers on these cards are development/placeholder data
  (already marked with a DEV badge), the line is a **seeded, deterministic
  decorative shape** ending in the direction the stated change % indicates
  — not a fabricated trading signal, and clearly a different thing from
  the real crypto sparklines above. Worth keeping that distinction in mind
  if you extend this further.

## New: atmospheric, centered hero (matching the reference feel)

Rebuilt `.hero` into `.hero-atmospheric`: a dark dusk-toned gradient sky
with two layered CSS `clip-path` mountain-silhouette bands (no photo used —
built from shapes, in the app's own navy/cyan/pink/orange palette so it
stays on-brand), centered composition, a script-font eyebrow ("Discover",
Dancing Script) above a large italic serif heading (Playfair Display,
via Google Fonts), and glass-style CTA buttons. The KSE-100 stat moved into
a small glass badge in the top-right corner instead of splitting the
layout, keeping the centered composition intact.

### Verified in this sandbox

Ran the full server against this exact packaged directory and confirmed
all 14 core endpoints return 200, including the new sparkline data path
on `/api/crypto/live`. Python and JS both compile clean.

---

## Fixed: the disappearing nav tabs (and "where's my Forex tab")

Root cause found: `.topnav-links` used `overflow-x: auto` with a **hidden
scrollbar**. As tabs accumulated across earlier rounds (17 of them), extras
silently scrolled off-screen with zero visual cue — that's exactly why
Forex "disappeared," and why browser zoom made it worse. Fixed by switching
the nav to wrap onto multiple rows instead, so every tab is always visible
regardless of zoom or window width.

## Fixed: logo sizing

Switched `.brand-mark img` to `object-fit: contain` at proper header
height, showing your actual logo cleanly instead of the earlier crop-to-icon
hack.

## New: green/red tint backgrounds

Added `.tint-positive` / `.tint-negative` (light green/red backgrounds, not
just text color) and applied them to index cards, multi-market cards,
commodity cards, portfolio rows, watchlist rows, and All-PSX-Stocks rows.

## New: Dashboard overhaul

- **"Today Across All Markets" is now the first section**, before the hero
  — PSX indices + world markets (S&P 500, Bitcoin, Gold, WTI, forex),
  every card tinted green/red by trend.
- **Sentiment By Market** section — PSX Fear & Greed ring plus a
  crypto/forex/commodities sentiment strip (reusing `non_equity_sentiment`
  from `/api/market360`).
- **Four live scrolling ticker strips** (PSX / Crypto / Forex / Mutual
  Funds) at the very top of the dashboard, CSS-animated marquees pulling
  from the same live endpoints as their respective tabs.
- **Quick Add to Watchlist** box replaces the old duplicate sentiment card.

## New: cross-asset Watchlist (stocks, crypto, forex, funds)

The `watchlist` table now has an `asset_type` column. You can add crypto,
forex pairs, and mutual funds to your watchlist — not just PSX stocks —
via the ★ icon that now appears on every row across All PSX Stocks, Crypto,
Forex, and Mutual Funds tables, or by typing a symbol directly on the
Watchlist page. The page has:
- **Sort** by change %, price, or name
- **Drag-to-reorder** (HTML5 drag and drop, persisted via
  `/api/watchlist/reorder`)
- **Remove** (× button)

**Bug caught and fixed during testing**: fund names were being
force-uppercased on add, which silently broke the price lookup for any
fund watchlist item (MUFAP fund names are mixed-case, e.g. "Meezan Cash
Fund"). Fixed with case-insensitive SQL matching (`COLLATE NOCASE`) across
add/remove/reorder — verified with a live add → lookup → mixed-case-URL
remove round trip.

## New: Portfolio upgrades

- **Calendar icon** (📅) instead of a "View Calendar" text link
- **Per-holding sparkline** column (real trend once `holding_daily` has 2+
  recorded days — same "builds up over time" pattern as elsewhere)
- **Explicit "% Profit" column** (previously only shown as small text
  under P/L)
- **Drag-to-reorder** and a **sort dropdown** (P/L, % Profit, Value),
  persisted via `/api/portfolio/reorder`

## New: All PSX Stocks — more inline detail

Added a **move filter** (Up today / Down today / Big gainers ≥5% / Big
losers ≤-5%), a **Volume** column, a **Held By Funds** column (honestly
limited to the ~5 stocks tracked in `TOP_STOCKS_THREE_WAYS` — see the note
in the table), and the ★ watchlist-add icon.

## New: Crypto tab — top strip + live sentiment

- Top-6 coins strip (tinted green/red) above the main table
- **Overall Crypto Sentiment** card — live Fear & Greed Index from
  alternative.me's free public API (no key required), with a gauge ring
  and a bar

## New: Mutual Funds — NAV history + sparkline

`record_fund_nav_history()` now saves each fund's NAV daily (hooked into
the existing MUFAP scrape), and the funds table has a **Trend** sparkline
column at the end of each row that fills in as history accumulates.

## New: World Clock — crypto & mutual fund notes

Added cards clarifying that crypto trades 24/7 (no exchange hours apply)
and that Pakistani mutual fund NAVs are struck once per business day by
each AMC, not continuously like stocks.

## New: Pakistan Profile — petrol & diesel

Added to `PAKISTAN_PROFILE`, clearly labeled as development values (no
verified free live API for OGRA's fortnightly fuel-price notifications).

### Verified in this sandbox

Ran the full server and confirmed:
- All ~15 core endpoints return 200
- Full multi-asset watchlist round trip (add stock/crypto/forex/fund →
  GET → reorder → remove) works correctly, including the mixed-case fund
  name fix
- Portfolio reorder endpoint works
- Python and JS both compile clean

---

## New: real 392-fund Pakistan mutual fund directory

Read directly from your uploaded MUFAP "Asset Allocation" export
(`MUFAP_FUND_DIRECTORY` in `app.py`) — 392 real open-end funds across 22
AMCs and 23 categories, with fund name, AMC, category, inception date and
AUM (July 2026, PKR millions). This is now:
- The fallback whenever MUFAP's live NAV scrape fails — you always see all
  392 real funds, not a placeholder list.
- Merged with live NAV/YTD when the scrape succeeds (matched by fund name),
  so you get real AUM + real live NAV together.
- The data source for the Compare Funds tool.

Verified: total AUM across all 392 funds sums to ~PKR 3.72 trillion, which
lines up with independent industry figures (~PKR 3.9–4.2T total mutual
fund AUM) — a good sanity check that the extraction is accurate.

Mutual Funds page now has search, a category filter (auto-populated from
the real 23 categories), and pagination (50/page).

## New: Commodities tab

Nav → **Commodities**. Gold, Silver, Platinum, WTI, Brent, Natural Gas,
Copper, Wheat, Corn, Cotton. Goes live automatically if
`MARKET_DATA_API_KEY` is set (metals/energy via Twelve Data); grains stay
on development data even then since they're not reliably on the free tier.
Each card shows a LIVE/DEV badge.

## New: Dashboard cross-asset strip

Dashboard now shows Major Forex (EUR/GBP/JPY/AUD/PKR), Top 5 Mutual Funds
by AUM, and a Commodities snapshot — one new `/api/dashboard-highlights`
endpoint bundles all three so the dashboard doesn't need three separate
calls.

### Verified in this sandbox

Ran the full server and confirmed, with real numbers:
- `/api/mutual-funds` returns exactly 392 funds, 23 categories, 22 AMCs,
  total AUM ≈ PKR 3.72 trillion
- `/api/dashboard-highlights` correctly surfaces the top 5 funds by AUM
  (money-market funds dominate, as expected)
- `/api/commodities/live` correctly returns all 10 commodities marked
  "Development value" when no API key is set
- `/api/tools` still returns the full fund list for Compare Funds, plus
  the Stock Market Calculator in the catalog
- All endpoints return HTTP 200; Python and JS both compile clean

---

## New: live prices, no API key required

Two new nav items, both using free public APIs that don't need a key:

### Crypto
Nav → **Crypto**. Top 500 cryptocurrencies by market cap, live from
CoinGecko's free public `/coins/markets` endpoint (`fetch_crypto_live` in
`app.py`) — price, 24h change, market cap, 24h volume, searchable, paginated
50/page. Cached 90 seconds. On any failure (rate limit, network issue) it
falls back to a small labeled development list — the page always says which
("🟢 Live from CoinGecko" vs "🟡 Development data").

This is **not literally all cryptocurrencies** — depending on how you count,
there are anywhere from ~17,000 actively-tracked tokens to tens of millions
of ever-created tokens (most abandoned/scam/dead). Top 500 by market cap
covers effectively all trading activity that matters; anything past that is
mostly inactive noise.

### Forex
Nav → **Forex**. Live ECB reference rates for every major/minor currency
against USD, from Frankfurter's free public API (`fetch_forex_live`) — no
key needed, refreshed hourly (rates are daily ECB reference rates, not
tick-by-tick). Frankfurter tracks ~30 currencies, which covers roughly 95%+
of actual forex trading volume; PKR isn't ECB-tracked so it's layered in
from a labeled fallback value. Search box included. Truly exotic/regional
currencies beyond the ECB's ~30 would need a different, usually paid,
provider (same "needs an API key" situation as world indices).

### Mutual Funds — now searchable
Pakistan Mutual Funds page got a search box (`fundSearch`) so you can filter
by name/category once MUFAP's live scrape returns the full ~110-fund
directory.

### Verified in this sandbox

This sandbox has no internet access, so I confirmed the *fallback path*
works correctly end-to-end: both `/api/crypto/live` and `/api/forex/live`
return HTTP 200, correctly detect the network failure, and cleanly serve
the labeled development data instead of crashing or returning an error
page. On your machine (with normal internet access), the same code path
will instead successfully reach CoinGecko/Frankfurter and show real live
data — I wasn't able to verify that specific network round-trip from here,
but it follows the exact same tested request/fallback pattern as your
already-working PSX and MUFAP scrapers.

---

## New: Real technical-indicator recording

Previously, RSI/MACD/moving-average filters were all marked "Soon" with no
path to turning on. Now they do:

- Every background price refresh (`refresh_bulk_quotes`) also saves each
  PSX stock's daily close/high/low/volume to a new `stock_price_history`
  table (`record_daily_prices`). This happens automatically — no setup.
- `compute_technicals(symbol)` calculates **real** SMA(20/50/100/200),
  EMA(20), RSI(14), MACD(12,26,9), OBV, and Golden/Death Cross from that
  recorded history — nothing fabricated. Each indicator returns `None`
  until it has enough history (RSI needs 15 trading days, MACD needs 35,
  the 200-day SMA needs 200), and the screener catalog shows live progress
  (e.g. *"Activating: 26/50 trading days recorded (52%)"*) instead of a
  flat "Soon".
- Verified end-to-end: seeded 25 days of synthetic uptrending price history
  for a test symbol and confirmed RSI correctly computed to 100.0 (accurate
  for an unbroken uptrend), SMA20 computed correctly, the catalog correctly
  flipped RSI/EMA20/SMA20/OBV to "LIVE" once their thresholds were met
  while SMA50/100/200/MACD/Golden-Cross correctly stayed "Activating", and
  a screener filter (`rsi_overbought`) correctly matched only that symbol.
- **All PSX Stocks tab** now shows P/E, RSI(14), and a Trend badge
  (Golden Cross / Bullish 50>200 / Bearish 50<200 / Death Cross) per stock,
  plus a banner showing how many trading days of history have been
  recorded so far.
- Screener filters added: RSI range, RSI overbought/oversold, above
  20/50/200-day SMA, Golden Cross, Death Cross, MACD bullish.

### What this does NOT cover

Bullish RSI *divergence*, MACD divergence, Bollinger Bands, ADX, VWAP,
support/resistance, breakout detection, and all Smart Money Concepts /
chart-pattern items (Order Blocks, FVG, BOS/CHoCH, Wyckoff, Cup & Handle,
etc.) still need either a proper pattern-recognition engine or intraday
tick data — those stay honestly marked unavailable rather than faked.

### Timeline, if you leave this running

| Indicator | Trading days needed | Roughly |
|---|---|---|
| RSI(14), EMA20, SMA20, OBV | 15–20 | ~1 month |
| MACD | 35 | ~7 weeks |
| SMA50 | 50 | ~10 weeks |
| SMA100 | 100 | ~5 months |
| SMA200, Golden/Death Cross | 200–201 | ~10 months |

A historical-data vendor with PSX coverage could backfill this instantly
instead of waiting — ask if you want help evaluating one.

---

## Fixed: Windows crash on startup/restart

If you saw:
```
OSError: [WinError 10038] An operation was attempted on something that is not a socket
```
This is a well-known Windows-only bug in Werkzeug's dev-server auto-reloader
(`use_reloader=True`, the default when `debug=True`). It spawns a second
watcher process that re-runs the whole script; combined with our background
bulk-price-refresh thread, that reliably triggers this crash on Windows when
the reloader restarts. **Fixed** by setting `use_reloader=False` in the
`app.run(...)` call at the bottom of `app.py`. You'll need to manually
restart `python app.py` after editing code (no more auto-reload-on-save),
but the server no longer crashes on its own.

## New: Stock Market Calculator

Tools → **Stock Market Calculator** (client-side, no data needed). One tool,
six modes, selected from a dropdown:
- **Profit / Loss** — buy/sell price, quantity, brokerage % each side
- **Break-Even Price** — minimum sell price to cover brokerage both ways
- **Target Price** — price needed for a desired % gain
- **Position Sizing** — risk-based share count from account size, risk %,
  entry and stop-loss
- **Average Cost** — new average cost/quantity after adding to a position
- **Capital Gains Tax Estimate** — flat-rate CGT estimate (rate is an input
  since PSX/FBR CGT rates depend on holding period and filer status — the
  tool tells you to confirm the current rate rather than assuming one)

## New: Remove a holding from your portfolio

Portfolio page → each row now has a **×** button. Clicking it asks for
confirmation, then calls `DELETE /api/portfolio/holding/<symbol>`, which
removes the holding, its transaction history, and its recorded daily
calendar (`holding_daily`) entirely. This is destructive and irreversible —
the frontend confirms before calling it, and the backend returns 404 if the
symbol isn't currently held.

---

## Run

```powershell
pip install -r requirements.txt
python app.py
```

Then open http://127.0.0.1:5000/

Optional, for live world indices/forex/gold (see below):
```powershell
set MARKET_DATA_API_KEY=your_twelvedata_key
python app.py
```

---

## Everything in this build

### Portfolio
- Holdings, transactions, watchlist
- **Per-holding calendar** — click any holding row to open a modal with a
  value chart and a day-by-day P/L calendar from your purchase date to
  today. Click any day to see cumulative P/L up to that date. Days before
  real daily tracking started are filled with a labeled straight-line
  **estimate** (`Est.` badge) between your purchase price and the first
  real snapshot — see `holding_history()` in `app.py`.
- Whole-portfolio P/L calendar and chart (unchanged from earlier versions)

### Markets (full Portfolio360-style long page)
Risk sentiment gauge, multi-market cards (KSE-100/S&P 500/Tadawul/Bitcoin/
Gold/WTI/forex), cross-asset signals & highlights, PSX movers, trending
stocks, 52-week highs/lows, "Who Is Buying Pakistan" fund-flow matrix,
insider activity, market sentiment history, non-equity sentiment, "Top
Stocks Three Ways", levels-to-play, seasonality, Pakistan macro profile,
news/announcements/payouts, and the market calendar.

### All PSX Stocks — now with live prices
`/api/stocks/live` concurrently fetches the full PSX symbol directory
(12 workers at a time), cached and auto-refreshed every 10 minutes by a
background thread (`start_bulk_refresh_thread`), plus a manual refresh
button. Price and Change % columns are shown in the directory table.

### Stock Screener (new)
Nav → **Screener**. Two parts:
1. **Available filters** (real, computed from PSX data): price range,
   change % range, P/E range, volume minimum, sector, 52-week position
   (near high / near low / mid), above LDCP. Add filters one at a time,
   press **Run Screener**, results come from `/api/screener/run`.
2. **Full checklist reference** (`/api/screener/catalog`, backed by
   `FILTER_CATALOG` in `app.py`) — every item from your uploaded master
   checklist (~140 metrics: valuation, growth, profitability, financial
   health, balance sheet, dividends, ownership, market performance,
   technical analysis, Smart Money Concepts, price-action patterns, risk
   management), organized by section. Items we can compute today are
   marked available; everything else (RSI, MACD, moving averages, Golden/
   Death Cross, Order Blocks, FVG, BOS/CHoCH, Wyckoff, chart patterns,
   Sharpe/Sortino, EPS/ROE/margins, insider ownership, analyst ratings,
   etc.) is shown with a plain-language reason it isn't computed yet
   (needs historical OHLCV, a financial-statements vendor, or an
   ownership-filings feed) — **never faked**. Fabricating a "Bullish RSI
   Divergence" hit on a real stock would be actively misleading for
   anyone trading real money on it, so those stay clearly labeled
   "not available" rather than showing a made-up signal.

### Stock Detail — Fundamentals
Every stock page now has a Fundamentals card: P/E (gauge), 1-year/YTD
change, LDCP, and honestly-labeled placeholders ("Needs data vendor") for
EPS, book value, dividend yield/history — see `get_fundamentals()`.

### Pakistan Mutual Funds (new)
Nav → **Mutual Funds**. `fetch_mufap_funds()` attempts to scrape MUFAP's
public NAV listing; on any parse failure it falls back to a small
development fund list, and the page always shows which source you're
looking at ("🟢 Live from MUFAP" vs "🟡 Development data").

### World Clock & Trading Sessions (new)
Nav → **World Clock**. 10 major stock exchanges (PSX, LSE, NYSE, NASDAQ,
Tokyo, Hong Kong, Shanghai, Sydney, Tadawul, Dubai) plus the 4 forex
sessions (Sydney/Tokyo/London/New York) — local time, open/closed status,
countdown to next open/close. Computed entirely client-side from IANA
timezones, DST-safe, no API key needed.

### Pre-market alert banner (new)
A dismissible banner (+ browser notification, if you grant permission)
appears when more tracked global markets are down than up ahead of PSX's
open. `compute_premarket_signal()` uses live vendor data when available,
falls back to development data otherwise, and always says which.

### Live world indices / forex / gold (optional)
Set the `MARKET_DATA_API_KEY` environment variable (Twelve Data's free
tier works — see `TWELVEDATA_SYMBOLS` in `app.py`) to pull real prices for
S&P 500, Bitcoin, Gold, WTI, and USD/CAD. Without a key, everything shows
clearly-labeled development values (a **DEV**/**LIVE** badge appears on
each multi-market card so it's always obvious which you're seeing).
Swap `MARKET_DATA_PROVIDER` and extend `fetch_live_index()` to use a
different vendor (Alpha Vantage, Finnhub, etc.) if you prefer.

### Journal & Tools
Journal (articles + podcasts) and Tools (Zakat Calculator, Dividend
Purification, FIRE Calculator, Goal Planner, SIP Calculator, Compare
Stocks — live PSX data — DCF Calculator, Compare Funds) — all working,
client-side calculators except Compare Stocks.

### Real logo & palette
`static/images/yalvon360-logo.png` is your actual logo. The palette
(navy `#182860`, cyan `#29B6E8`, pink `#EC1876`, orange `#FBA53C`) is
drawn directly from it and used throughout.

---

## What still needs your input to go fully live

| Feature | What's needed |
|---|---|
| Live world indices/forex/gold | A `MARKET_DATA_API_KEY` (see above) |
| Real historical P/L backfill (per-holding calendar) | A licensed PSX historical-price feed, wired into `holding_history()` |
| Technical Analysis / Smart Money Concepts filters (RSI, MACD, Order Blocks, etc.) | Multi-year historical OHLCV data + an indicator engine |
| Deep fundamentals (EPS, ROE, margins, DCF, etc.) | A financial-statements data vendor |
| Ownership/insider data per stock | A PSX filings feed |
| True pre-open detection | Live data polled a few minutes before PSX's 9:15 AM PKT open |
| MUFAP scrape reliability | MUFAP's page markup can change; monitor `fetch_mufap_funds()` |

## Market-data usage note

Before public/commercial deployment, obtain the appropriate PSX
market-data rights/license and replace the development data blocks in
`app.py` with an authorized feed. The bulk PSX fetch (`/api/stocks/live`)
is polite by design (limited concurrency, 10-minute refresh interval) but
was still built for personal/development use, not high-frequency
commercial redistribution.
