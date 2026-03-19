# Morning Portal — Design Spec

**Date:** 2026-03-19
**Status:** Approved

---

## Overview

A personal morning dashboard served as a web page, designed for iPhone and iPad. Displays four data widgets — weather, bus departures, electricity prices, and Swedish news — each fetched independently in parallel so a slow source never blocks the rest.

Deployed as a single Docker container on a Rock 5B home server. Same image runs on local Docker Desktop for development.

---

## Architecture

### Single Express.js container

One Node.js/Express app replaces the current nginx setup. It serves the static frontend (`public/index.html`) and exposes four API routes that proxy external data sources server-side. API keys never reach the browser.

```
/
├── server.js            # Express: static serving + /api/* routes
├── package.json
├── Dockerfile           # node:alpine
├── docker-compose.yml   # port 3000:3000, SL_API_KEY env var
└── public/
    └── index.html       # Frontend: parallel widget fetching
```

### API routes

| Route | External source | Auth |
|---|---|---|
| `GET /api/weather` | SMHI open data | None |
| `GET /api/bus` | Trafiklab ResRobot v2.1 | `SL_API_KEY` env var |
| `GET /api/electricity` | elprisetjustnu.se | None |
| `GET /api/news` | RSS: SVT, DN, Aftonbladet | None |

All outbound fetch calls use a **8-second timeout**. If a call times out or errors, the API route returns `{ error: "source unavailable" }` with HTTP 200 (so the frontend handles it as a widget-level error, not a page-level failure).

### Portability

`SL_API_KEY` is set as an environment variable in `docker-compose.yml`. The same Docker image works on both Mac Docker Desktop and the Rock 5B ARM server (`node:alpine` is multi-arch).

---

## Frontend

### Layout

- **Mobile-first:** Single column on iPhone, 2-column grid on iPad (`768px` breakpoint).
- **Dark theme** throughout — easy on the eyes at wake-up time.
- **Header:** Current time (large) and date, updated every second via JS.
- **Auto-refresh:** Full data refresh every 10 minutes via `setInterval`. On refresh failure, the widget retains its previous data and shows a small "last updated HH:MM" timestamp — it does not replace data with an error message mid-session.

### Widgets (top to bottom on mobile)

1. **Rain warning banner** — full-width, amber/red. Shown when any hour between 06:00–22:00 today has `Wsymb2` value of 8–27 (light drizzle through heavy snow), or `pmean >= 0.5` mm/h. Always at the top when visible.
2. **Weather** — current conditions (derived from current hour's `Wsymb2`), today's high/low (max and min `t` parameter across all forecast hours from 00:00–23:00 local time today in `Europe/Stockholm`), and a plain-text precipitation summary for daytime hours (06:00–22:00).
3. **Bus departures** — next 3–5 departures from Strömma Kanal → Slussen. Shows line number, destination string, and minutes until departure. Any departure ≤5 minutes away is highlighted red.
4. **Electricity prices** — bar chart (06:00–22:00), bars colored green (<0.50 SEK/kWh) → yellow (0.50–1.00) → red (>1.00). Current hour bar is outlined/highlighted. Powered by **Chart.js v4** (loaded from CDN: `https://cdn.jsdelivr.net/npm/chart.js@4`).
5. **News** — 5–8 headlines from Swedish sources (SVT, DN, Aftonbladet), sorted by publication date descending. Each item shows source label, headline, and 1–2 sentence brief. **Deduplication:** articles whose headlines share 5 or more consecutive words are considered duplicates; only the earliest-published copy is kept.

### Loading states

Each widget renders a skeleton/spinner independently while its `/api/*` call is in flight. Widgets render as data arrives.

### Error states (initial load only)

If an API call fails on first load, the widget shows an inline `"Could not load [widget name]"` message. Other widgets are unaffected.

---

## Data Sources

### Weather — SMHI

- Free, no API key.
- Endpoint: `https://opendata-download-metfcst.smhi.se/api/category/pmp3g/version/2/geotype/point/lon/18.0965/lat/59.3260/data.json`
  - Coordinates: lat **59.3260**, lon **18.0965** (Djurgårdskanalen / Strömma Kanal ferry stop area, Stockholm).
- Key parameters in response (`timeSeries[].parameters[]`):
  - `t` — air temperature (°C)
  - `pmean` — mean precipitation (mm/h)
  - `Wsymb2` — weather symbol (integer 1–27; 1=clear, 8–27 = precipitation of various types)
- High/low: filter `timeSeries` to entries where the `validTime` falls on today's date in `Europe/Stockholm` (00:00–23:59). Take max and min `t` values.
- Rain warning: check entries where `validTime` is between 06:00–22:00 local. Flag if any has `Wsymb2 >= 8` or `pmean >= 0.5`.
- `precipSummary`: group consecutive rainy hours (where `Wsymb2 >= 8` or `pmean >= 0.5`) into time ranges, e.g. "Rain expected 10:00–14:00, 17:00–19:00". Show up to two ranges; if more, append "and later". Return empty string if no rain.

**`/api/weather` response shape:**
```json
{
  "symbol": 3,
  "tempNow": 8.2,
  "tempHigh": 12.1,
  "tempLow": 4.3,
  "rainWarning": true,
  "precipSummary": "Rain expected 10:00–14:00"
}
```

---

### Bus — Trafiklab ResRobot v2.1

- Requires `SL_API_KEY` env var.
- **Stop ID lookup:** Call `GET https://api.resrobot.se/v2.1/location.name?input=Str%C3%B6mma+Kanal&key={key}&format=json` and use the `id` field from the first result (stop-level ID, not platform ID). Cache this ID in memory at server startup — do not re-fetch on every request.
- **Departures:** `GET https://api.resrobot.se/v2.1/departureBoard?id={stopId}&maxJourneys=10&key={key}&format=json`
- **Direction filter:** Keep only departures where the `direction` field (case-insensitive) contains `"slussen"` or destination stop name contains `"Slussen"`. If no results match after filtering, return all departures unfiltered (fail-open, since route data may vary).
- Return the first 5 matching departures.

**`/api/bus` response shape:**
```json
{
  "departures": [
    {
      "line": "80",
      "direction": "Slussen",
      "scheduledTime": "07:42",
      "minutesUntil": 4
    }
  ]
}
```

---

### Electricity — elprisetjustnu.se

- Free, no API key.
- Construct today's date in **`Europe/Stockholm`** timezone to avoid midnight UTC/local mismatch.
- Endpoint: `https://www.elprisetjustnu.se/api/v1/prices/{year}/{MM}-{DD}_SE4.json`
- Response is an array of hourly objects with `SEK_per_kWh` and `time_start`.
- Filter to hours 06:00–22:00 local time only.

**`/api/electricity` response shape:**
```json
{
  "hours": [
    { "hour": 6, "price": 0.43 },
    { "hour": 7, "price": 0.61 }
  ],
  "currentHour": 7
}
```

---

### News — RSS feeds

- Sources:
  - SVT Nyheter: `https://www.svt.se/nyheter/rss.xml`
  - DN: `https://www.dn.se/rss/`
  - Aftonbladet: `https://rss.aftonbladet.se/rss2/small/pages/sections/senastenytt/`
- Parse with `rss-parser` npm package. Merge all items, sort by `pubDate` descending.
- **Deduplication:** Sort all items by `pubDate` ascending before deduplication. Remove any item whose title shares 5+ consecutive words with an already-seen item — this keeps the item with the earliest `pubDate` (the original source) and discards later duplicates. After deduplication, re-sort descending for display.
- Return top 8 after deduplication.
- `brief` is populated from the RSS item's `contentSnippet` or `content` field, truncated to 200 characters.

**`/api/news` response shape:**
```json
{
  "articles": [
    {
      "title": "Headline here",
      "brief": "Short description...",
      "source": "SVT",
      "url": "https://...",
      "pubDate": "2026-03-19T06:30:00Z"
    }
  ]
}
```

---

## Docker

### Dockerfile
```dockerfile
FROM node:alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/ || exit 1
CMD ["node", "server.js"]
```

### docker-compose.yml
```yaml
version: "3.8"
services:
  morning-portal:
    build: .
    ports:
      - "3000:3000"
    restart: unless-stopped
    environment:
      - SL_API_KEY=${SL_API_KEY}
```

A `.env` file (gitignored) holds `SL_API_KEY=<value>` for local development. On the Rock 5B, the env var is set via a `.env` file on the server.

---

## Out of Scope

- Authentication / access control
- Push notifications
- Historical electricity data
- Multiple bus routes or stops
- User-configurable settings UI
