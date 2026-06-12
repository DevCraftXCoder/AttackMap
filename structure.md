# AttackMap — Architecture Reference

AttackMap is a real-time auth-failure geo-dashboard running entirely on Cloudflare's edge. It ingests failed login attempts from an API server, stores them in D1, and renders them through a seven-view Next.js 15 dashboard connected via Server-Sent Events.

---

## Data Pipeline

```
1. Ingest
   POST /ingest receives auth-failure event
   → validated, written to D1 auth_failures table
   → 202 returned immediately (low-latency write path)

2. Async Enrichment (waitUntil)
   → Geo resolution: Zippopotam + U.S. Census fallback
     → result cached in Workers KV (geo: prefix)
   → Anomaly scoring: 0–100 score assigned
     → pattern classified (brute_force / credential_stuffing /
        rate_probe / tor_exit / known_bad_ip / bot / geo_concentration)
   → Threat feed check: IP checked against Spamhaus DROP + CINS Army
   → Alert evaluation: active alert rules checked against new event
   → KV dedup key written (SSE dedup: prefix)

3. Storage (D1 — SQLite at the edge)
   Tables:
   ├── auth_failures    (events, geo, anomaly_score, pattern, route, reason)
   ├── ip_blocks        (ip, expiry, created_at)
   ├── country_blocks   (country_code, expiry, created_at)
   └── alert_rules      (type, threshold, window_seconds, enabled)

4. Streaming
   GET /events — SSE endpoint
   → polls D1 for rows newer than last-seen ID
   → pushes JSON-encoded events to all connected clients
   → dedup via Workers KV (prevents duplicate on reconnect)

5. Dashboard (Next.js 15, App Router)
   → receives SSE stream
   → renders seven views (see below)
   → state managed via React context + useReducer
   → framer-motion for animated transitions between views
```

---

## Dashboard Views

| View | Components | Primary Data Source |
|------|-----------|-------------------|
| **Map** | CommandMap, LeafletMap, CobeGlobe, ViewSwitcher | D1 + SSE stream |
| **Log** | EventTable, FilterBar, AnomalyBadge, IPProbeDrawer | SSE stream |
| **Intel** | ThreatSummary, ThreatFeedPanel, TopCountries, TopRoutes | D1 aggregate + KV cache |
| **Stats** | AttackHeatmap, TimeSeriesChart, RouteBreakdown, ReasonChart | D1 aggregate |
| **Blocklist** | BlockTable, AddBlockForm, ExpiryPicker, CountryBlockToggle | D1 |
| **Rules** | RuleList, AddRuleForm, RuleTypeSelector, ThresholdInput | D1 |
| **Replay** | ReplayScrubber, ReplayMap, SpeedControl, FrameCounter | D1 window query |

Total UI surface: approximately 30 React components, ~9,500 lines of TypeScript/TSX.

---

## Backend Route Groups

```
/ingest
  POST /ingest                  — write auth failure event

/events
  GET  /events                  — SSE stream of new events

/api/blocks
  GET    /api/blocks            — list active IP + country blocks
  POST   /api/blocks/ip         — add IP block (optional expiry)
  DELETE /api/blocks/ip/:ip     — remove IP block
  POST   /api/blocks/country    — add country block (optional expiry)
  DELETE /api/blocks/country/:cc — remove country block

/api/rules
  GET    /api/rules             — list alert rules
  POST   /api/rules             — create rule
  PUT    /api/rules/:id         — update rule
  DELETE /api/rules/:id         — delete rule

/api/probe
  GET  /api/probe/:ip           — AbuseIPDB enrichment + reverse lookup

/api/share
  POST /api/share               — generate share token
  GET  /api/share/:token        — resolve token → aggregate stats (public)

/api/export
  GET  /api/export              — CSV or JSON export (up to 10k rows)

/api/backfill
  POST /api/backfill            — geo-resolve historic events missing coordinates

/api/intel
  GET  /api/intel/summary       — automated threat analysis (KV-cached 5 min)
```

All state-mutating endpoints require `Authorization: Bearer <ADMIN_TOKEN>`. `/api/share/:token` is public (read-only, aggregate only).

---

## Security Architecture

### Request-level enforcement

A Hono middleware layer runs on every Worker request before any route handler:

1. **IP block check** — looks up requesting IP in Workers KV (D1 fallback on miss). Blocked IPs receive `403` immediately.
2. **Country block check** — reads `CF-IPCountry` header; blocked country codes receive `403`.
3. **Admin auth** — all `/api/*` routes validate `Authorization: Bearer` via Web Crypto `timingSafeEqual`.
4. **CSRF token** — state-mutating endpoints (`POST`, `PUT`, `DELETE`) require a valid CSRF token.

### Data privacy

- The SSE stream delivers full event records only to authenticated admin sessions.
- Share tokens expose aggregate statistics only — no raw IPs, no individual events.
- AbuseIPDB lookups are proxied through the Worker; the API key is never sent to the browser.
- KV dedup keys use a salted hash of the event ID, not the raw IP, to prevent enumeration.

### Auth-fail logging hardening

- Loud alert cooldown: the same IP + route combination triggers at most one alert per configurable window.
- Salt on auth-fail KV keys prevents KV-based IP enumeration across the dedup namespace.
- Share tokens validated on every read — expired tokens return `410 Gone`.

---

## Workers KV Namespaces

| Prefix | Purpose | TTL |
|--------|---------|-----|
| `geo:<ip>` | Cached geo coordinates from Zippopotam/Census | 24 hours |
| `dedup:<hash>` | SSE event dedup key | 60 seconds |
| `block:ip:<ip>` | IP block fast-path cache | Matches block expiry |
| `block:cc:<code>` | Country block fast-path cache | Matches block expiry |
| `intel:summary` | Automated threat analysis cache | 5 minutes |

---

## Stack Summary

| Layer | Technology |
|-------|-----------|
| Edge runtime | Cloudflare Workers |
| Database | D1 (SQLite at the edge) |
| KV store | Workers KV |
| Backend framework | Hono (TypeScript) |
| Frontend | Next.js 15 (App Router) |
| Map — interactive | Leaflet |
| Map — 3D | Cobe (WebGL) |
| Charts | recharts |
| Animation | framer-motion |
| Streaming | Server-Sent Events |
| IP enrichment | AbuseIPDB |
| Threat feeds | Spamhaus DROP, CINS Army |
