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
| `GET /api/bus` | Trafiklab ResRobot v3 | `SL_API_KEY` env var |
| `GET /api/electricity` | elprisetjustnu.se | None |
| `GET /api/news` | RSS: SVT, DN, Aftonbladet | None |

### Portability

`SL_API_KEY` is set as an environment variable in `docker-compose.yml`. The same Docker image works on both Mac Docker Desktop and the Rock 5B ARM server (`node:alpine` is multi-arch).

---

## Frontend

### Layout

- **Mobile-first:** Single column on iPhone, 2-column grid on iPad (`768px` breakpoint).
- **Dark theme** throughout — easy on the eyes at wake-up time.
- **Header:** Current time (large) and date, updated every second via JS.
- **Auto-refresh:** Full data refresh every 10 minutes via `setInterval`.

### Widgets (top to bottom on mobile)

1. **Rain warning banner** — full-width, amber/red, shown only when precipitation is forecast for today. Always at the top.
2. **Weather** — current conditions icon/description, today's high/low, hourly precipitation summary for daytime hours.
3. **Bus departures** — next 3–5 departures from Strömma Kanal → Slussen. Shows line, destination, minutes until departure. Departure highlighted red if <5 minutes away.
4. **Electricity prices** — bar chart for 06:00–22:00, bars colored green→yellow→red by price level. Current hour highlighted. Powered by Chart.js.
5. **News** — 5–8 headlines from Swedish sources (SVT, DN, Aftonbladet), each with source label and 1–2 sentence brief.

### Loading states

Each widget renders a skeleton/spinner independently while its `/api/*` call is in flight. Widgets render as data arrives — a slow news feed doesn't delay weather or bus data.

### Error states

Each widget shows an inline error message if its API call fails, without affecting the other widgets.

---

## Data Sources

### Weather — SMHI
- Free, no API key required.
- Use SMHI's open forecast API: `https://opendata-download-metfcst.smhi.se/api/category/pmp3g/version/2/geotype/point/lon/{lon}/lat/{lat}/data.json`
- Coordinates for Strömma Kanal area (Stockholm): lat ~59.32, lon ~18.09 (to be confirmed).
- Parse `pcat` (precipitation category) and `pmean` (precipitation mean) parameters to determine rain likelihood.
- Parse `t` (temperature) for high/low, `Wsymb2` for weather symbol.

### Bus — Trafiklab ResRobot
- Requires `SL_API_KEY` (user has key).
- ResRobot Departures v3: `https://api.resrobot.se/v2.1/departureBoard?key={key}&id={stopId}&maxJourneys=5&format=json`
- Stop ID for "Strömma Kanal" to be looked up via ResRobot Stop Lookup API during implementation.
- Filter results to show only departures toward Slussen.

### Electricity — elprisetjustnu.se
- Free, no API key required.
- `https://www.elprisetjustnu.se/api/v1/prices/{year}/{month}-{day}_SE4.json`
- Returns hourly prices in SEK/kWh. Filter to 06:00–22:00 for display.

### News — RSS feeds
- SVT Nyheter: `https://www.svt.se/nyheter/rss.xml`
- DN: `https://www.dn.se/rss/`
- Aftonbladet: `https://rss.aftonbladet.se/rss2/small/pages/sections/senastenytt/`
- Parse server-side with `rss-parser` npm package. Merge and sort by date, return top 8.

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
CMD ["node", "server.js"]
```

### docker-compose.yml
```yaml
services:
  morning-portal:
    build: .
    ports:
      - "3000:3000"
    restart: unless-stopped
    environment:
      - SL_API_KEY=${SL_API_KEY}
```

A `.env` file (gitignored) holds `SL_API_KEY=<value>` for local development. On the Rock 5B, the env var is set directly in the compose file or via a `.env` file on the server.

---

## Out of Scope

- Authentication / access control
- Push notifications
- Historical electricity data
- Multiple bus routes or stops
- User-configurable settings UI
