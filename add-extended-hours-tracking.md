# APEX — Add Pre-Market & Post-Market Tracking

## Objective
Add extended-hours (pre-market and post-market) price tracking to the APEX app without breaking any existing functionality.

## Constraints
- **Zero regressions**: All existing watchlist, dashboard, ticker bar, and stage classification logic must remain untouched.
- **Additive only**: New code should be in separate modules/functions. Do not modify existing data-fetching or rendering functions — extend them.
- **Graceful degradation**: If extended-hours data is unavailable (API failure, weekend, holiday), the UI should silently fall back to showing regular-hours data only. No errors, no broken layout.

## Architecture

### 1. Data Layer — `extendedHours.js` (new module)

Create a new module that:
- Exports `fetchExtendedHoursQuote(ticker)` → returns `{ preMarket: { price, change, changePct }, postMarket: { price, change, changePct }, source, timestamp }`
- Only fetches when market is CLOSED (use the existing market-status logic already in the app)
- During pre-market hours (4:00–9:30 AM ET): fetch pre-market data
- During post-market hours (4:00–8:00 PM ET): fetch post-market data
- Outside extended hours: return `null` gracefully
- Implements a simple cache (60-second TTL) to avoid hammering the API
- **Primary: Polygon.io** (user has API key already in the project)
  - Snapshot endpoint: `https://api.polygon.io/v2/snapshot/locale/us/markets/stocks/tickers/{TICKER}?apiKey={KEY}`
    - Returns `ticker.preMarket`, `ticker.afterHours`, and `ticker.day` in a single call
    - This is the most efficient option — one call per ticker gets everything
  - Alternative for granular data: `https://api.polygon.io/v2/aggs/ticker/{TICKER}/range/1/minute/{date}/{date}?adjusted=true&sort=desc&limit=5&apiKey={KEY}`
  - Polygon has full CORS support for browser requests — no proxy needed
- **Find the existing Polygon API key** in the codebase (likely in a config, .env, or constants file) and reuse it. Do NOT create a second key variable.
- Add error handling with try/catch on every fetch — never let a failed extended-hours call crash the app
- Respect Polygon rate limits (5 calls/min on free tier, higher on paid). The 60-second cache TTL handles this naturally for small watchlists, but add a request queue if the watchlist exceeds 5 tickers.

### 2. Ticker Bar Enhancement

In the top ticker bar (SPY, QQQ, IWM, DIA, VIX, DXY, OIL):
- When extended-hours data is available, append a small badge AFTER the existing RTH change display
- Format: `PM: +0.34%` or `AH: -0.12%` in a muted/dimmed style
- Use a slightly smaller font size than the main ticker text
- Color: green/red matching existing app palette, but at ~60% opacity to visually distinguish from RTH data
- If no extended-hours data → show nothing (no empty badge, no placeholder)

### 3. Watchlist Table — Optional Column

- Add a toggle button near the existing "Refresh" button labeled "EXT HRS" (off by default)
- When toggled ON: insert a new column after PRICE called "EXT" showing:
  - Pre-market price + change% during pre-market hours
  - Post-market price + change% during post-market hours
  - "—" when no extended data available
- When toggled OFF: column is completely removed from DOM (not just hidden)
- The toggle state should persist in localStorage
- Column styling should match existing table column styles exactly

### 4. Timing Logic

```
Pre-Market:  Mon–Fri, 4:00 AM – 9:29 AM ET
RTH:         Mon–Fri, 9:30 AM – 4:00 PM ET  
Post-Market: Mon–Fri, 4:01 PM – 8:00 PM ET
Weekend/Holiday: No extended-hours data
```

- Use the app's existing timezone/market-status detection if available
- If not, create a simple `getMarketSession()` utility returning `'pre' | 'rth' | 'post' | 'closed'`

### 5. Auto-Refresh

- During extended hours, auto-refresh extended-hours data every 60 seconds
- Use `setInterval` with proper cleanup (clear interval when session changes or component unmounts)
- Do NOT interfere with the existing refresh logic for RTH data

## Testing Checklist

Before considering this done, verify:

- [ ] Regular market hours: App behaves exactly as before. No visual changes. No console errors.
- [ ] Pre-market session: Ticker bar shows PM badges. Watchlist EXT column (if toggled) shows pre-market data.
- [ ] Post-market session: Ticker bar shows AH badges. Watchlist EXT column shows post-market data.
- [ ] Weekend: No extended-hours UI elements visible. No failed API calls in console.
- [ ] API failure: Simulate by disconnecting network — app continues working with RTH data only.
- [ ] Toggle persistence: Enable EXT HRS toggle, refresh page — toggle state preserved.
- [ ] Stage classification: Verify MRVL, ONTO, WMT, TGT still show correct Stage 2A classifications.
- [ ] Performance: No visible lag from additional API calls. Check Network tab for call frequency.

## Files to NOT Modify

Identify and list (do not edit) these existing files:
- Stage classification engine / scoring logic
- Watchlist CRUD (add/edit/remove ticker)
- Dashboard layout and routing
- Any CSS that affects the stage badge styling

## File Structure

```
src/
├── extendedHours.js          ← NEW: data fetching + cache
├── extendedHoursUI.js        ← NEW: ticker bar badges + table column
├── marketSession.js          ← NEW: session detection utility
└── ... (existing files untouched)
```

## Summary

This is a **purely additive** feature. The mental model: extended-hours tracking is a transparent overlay that activates only when relevant and disappears silently when not. If you're ever unsure whether a change could break existing behavior, err on the side of isolating it in the new modules.
