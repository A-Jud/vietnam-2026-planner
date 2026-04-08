# Technical Changelog & Decision Log

## Recovery Points

| Commit | Tag | Description |
|--------|-----|-------------|
| `25e9dd8` | Pre-review baseline | Original V6_1 — before any code review changes |
| `c00cead` | Google Drive + iframe | OAuth hardcoded, route iframe panel, initGIS timing fix |
| `fe3f4f3` | XSS batch 1 | `esc()` helper applied to all innerHTML interpolations |
| `5516971` | Security batch 2 | API key header, iframe sandbox, JSON try-catch, CSP |
| `1d4bbf5` | Security batch 3 | `escAttr()` for onclick, URL validation, JSON import validation |
| `9bef86b` | Route comparison + history | Google Maps version toggle, history creation on all saves |

To recover to any point: `git checkout <commit-hash>` (read-only) or `git reset --hard <commit-hash>` (destructive).

---

## Session 1 — 2026-04-08/09

### Google Drive OAuth Integration
**Decision:** Hardcode the Google OAuth Client ID (`449350453148-...`) directly in `Config.CLIENT_ID` rather than requiring each user to enter their own.
**Why:** Better UX — any user can connect their Google Drive with one click. The `drive.file` scope only accesses files the app itself created.
**Trade-off:** Client ID is visible in source code. Acceptable because it's an OAuth *client* ID (not a secret), and the scope is minimal.

### Google Maps in iframes
**Decision:** Use the legacy `maps.google.com/maps?...&output=embed` format instead of the Google Maps Embed API.
**Why:** The Embed API requires a paid API key. The legacy embed format works without any key.
**Limitation:** Google Maps blocks `X-Frame-Options` on non-embed URLs. Solution: Google links open in new tabs; only embed URLs load in iframes.

### Route iframe panel (left map area)
**Decision:** Show Google Maps routes in a panel overlaying the left-side Leaflet map, with a "Open in Google Maps" button.
**Why:** Users want to see routes in-app without leaving the page. The button handles the "More options" limitation (Google blocks non-embed URLs in iframes).

### initGIS timing race condition
**Problem:** `<script async>` for Google Identity Services could fire `onload="initGIS()"` before the `<script type="module">` set `window.initGIS`.
**Fix:** Changed from anonymous `window.initGIS = function()` to named `function initGIS()` + `window.initGIS = initGIS` in module scope. Added fallback: `if (typeof google !== 'undefined' && google.accounts) initGIS()` at module end.

### XSS Prevention — `esc()` helper
**Decision:** Created `esc(s)` using `textContent`/`innerHTML` DOM trick for HTML escaping. Applied to all 49 interpolation points in innerHTML.
**Scope:** AI results panel (options, providers, food, hotels, tips), timeline render (day title, desc, stay, links, options, tips, providers, transport), trips library, relocate dropdown.
**Why not DOMPurify?** Single-file app — adding a library dependency for escaping is overkill when a 1-line function covers all cases.

### XSS Prevention — `escAttr()` helper
**Decision:** Created `escAttr(s)` for onclick attribute context escaping (backslashes, quotes, backticks, angle brackets).
**Why:** The original code only escaped single quotes (`.replace(/'/g, "\\'")`), which is insufficient — backslash sequences and backticks could break out.

### Gemini API key moved to header
**Decision:** Changed from `?key=${key}` URL parameter to `x-goog-api-key` request header.
**Why:** URL parameters appear in browser history, network logs, and proxy caches. Headers do not.

### iframe sandboxing
**Decision:** Added `sandbox="allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox"` to both iframes.
**Why:** Prevents loaded pages from accessing parent DOM, navigating the top window, or running unrestricted scripts.

### Content Security Policy
**Decision:** Added CSP via `<meta http-equiv="Content-Security-Policy">` restricting:
- `script-src`: self, inline, eval (Tailwind needs these), unpkg, CDN, Google accounts
- `connect-src`: Gemini API, OpenAI API, Google Drive API, Nominatim
- `frame-src`: maps.google.com, www.google.com
**Trade-off:** `unsafe-inline` and `unsafe-eval` are needed because Tailwind CSS CDN uses them. A build step would eliminate this.

### JSON import validation
**Decision:** Validate `allTrips` is array and each trip has `days` array before applying state.
**Why:** Malformed JSON files could crash the app or inject unexpected data structures.

### History versioning — save on all modifications
**Problem:** `saveToHistory()` was only called on AI replans, not on individual saves (tips, options, providers, hotels).
**Fix:** Added `saveToHistory()` before mutation in `saveTip`, `saveGeneratedOption`, `saveGeneratedProvider`, `saveHotelSuggestion`.
**Trade-off:** More history entries = more localStorage usage. Acceptable for a trip planner.

### History — legacy migration fix
**Problem:** `migrateAndApplyState` set `history: []` for legacy data instead of preserving existing history.
**Fix:** Changed to `history: data.tripState?.history || []`. Also added `State.tripState.history = State.tripState.history || []` after all load paths.

### Route history comparison in Google Maps
**Decision:** When a day/trip comparison is active, the route iframe panel shows a toggle ("מסלול נוכחי" / "גרסת עבר") to switch between current and historical route embeds.
**Implementation:** `buildRouteUrlForDay(day)` extracted as reusable helper. `getHistoricalDayData(dayIdx)` checks both day-level and trip-level comparisons. Toggle tabs switch the iframe `src`.

### `openInIframe` URL validation
**Decision:** Block non-HTTP URLs from loading in iframes.
**Implementation:** `if (!/^https?:\/\//i.test(url)) return;` — prevents `javascript:`, `data:`, and other dangerous URI schemes.
