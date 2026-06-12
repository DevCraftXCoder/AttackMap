<div align="center">

> **Authorized Use Only** — This dashboard is designed for monitoring systems you own or have explicit written authorization to observe. Unauthorized access to systems is illegal under the CFAA and equivalent laws globally.

# AttackMap

**Real-time auth-failure geo-dashboard on Cloudflare Workers.**

[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6.svg)](https://www.typescriptlang.org/)
[![Cloudflare Workers](https://img.shields.io/badge/Cloudflare-Workers-F38020.svg)](https://workers.cloudflare.com/)
[![D1](https://img.shields.io/badge/D1-SQLite%20at%20the%20Edge-F38020.svg)](https://developers.cloudflare.com/d1/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

</div>

## Overview

AttackMap ingests failed authentication attempts from your API layer, enriches them with geo and reputation data, and renders the results as a live dashboard — entirely on Cloudflare's edge. Every component runs as a Cloudflare Worker or D1 query: no separate servers, no cold starts, no external databases.

The dashboard exposes seven views covering geo visualization, raw event log, threat intelligence, analytics, blocklist management, alert rules, and attack replay. Blocking rules written in the dashboard are enforced at the Worker edge on every subsequent request.

---

## Architecture

```
 Auth failure event
        │
        ▼
 ┌──────────────────┐
 │  Hono CF Worker  │  ← ingestion endpoint
 │  (TypeScript)    │
 └────────┬─────────┘
          │  write + waitUntil enrich
          ▼
 ┌──────────────────┐        ┌──────────────────┐
 │   D1 (SQLite)    │        │  Workers KV       │
 │  auth_failures   │        │  geo cache        │
 │  ip_blocks       │        │  threat summary   │
 │  alert_rules     │        │  SSE dedup        │
 └────────┬─────────┘        └──────────────────┘
          │  SSE stream
          ▼
 ┌──────────────────┐        ┌──────────────────┐
 │  Next.js 15 UI   │◄───────│  AbuseIPDB        │
 │  ~30 components  │        │  IP reputation    │
 │  7 dashboard views│        └──────────────────┘
 └──────────────────┘
          │
          ▼
   Blocking rules enforced
   by Worker middleware on
   every incoming request
```

**Key flows:**

- `POST /ingest` — writes the failure to D1; fires `waitUntil` tasks for geo resolution, anomaly scoring, alert evaluation, and KV dedup
- `GET /events` — SSE endpoint; polls D1 and pushes new rows to all connected dashboards
- Geo coordinates resolved via Zippopotam + U.S. Census fallback; results cached in Workers KV
- Block enforcement runs as Hono middleware — no external firewall or WAF dependency

---

## Features

### Geo Visualization
Three map modes selectable at runtime:
- **Command canvas** — custom SVG world map with animated attack arcs and real-time event pulse
- **Leaflet map** — interactive tile map with country-level marker clustering
- **Cobe 3D globe** — WebGL rotating globe with country dots colored by attack density

### Live Event Feed (SSE)
Server-Sent Events pipeline delivers new failures within seconds of ingestion. Anomaly scoring fires inline on each incoming event. The log view supports filtering by country, route, failure reason, and anomaly tier.

### Anomaly Scoring Engine
Each event receives a 0–100 anomaly score on ingestion. The engine classifies patterns as:
`brute_force` · `credential_stuffing` · `rate_probe` · `tor_exit` · `known_bad_ip` · `bot` · `geo_concentration`

Scores are stored in D1 and surfaced across all views.

### IP Deep Probe
One-click enrichment for any suspicious IP: AbuseIPDB confidence score, abuse category tags, reverse-lookup hostname, country, and ISP. Results render inline without leaving the dashboard.

### IP & Geo Blocking
- Block a single IP with optional expiry
- Block an entire country with optional expiry
- All blocks written to D1 and enforced by Hono middleware on every Worker request — sub-millisecond enforcement with no external dependency

### Alert Rules (CRUD)
Define threshold-based alert rules directly in the dashboard:
- Brute-force threshold (N failures in T seconds from one IP)
- Anomaly score threshold (fire when score exceeds N)
- Rules persist in D1; the ingestion Worker evaluates them on every event

### Attack Heatmap
Day-of-week × hour grid showing attack frequency density. Built with recharts. Supports per-country and per-route filters.

### Threat Intelligence Summary
Automated threat analysis of recent events: attack pattern classification, top targeted routes, top source countries, and recommended mitigations. Cached in Workers KV for five minutes to avoid re-running on every page load.

### Share Tokens
Generate a time-limited public link exposing aggregate statistics (event count, top countries, top targeted routes) — no raw IPs, no individual events, no admin access. Tokens expire automatically.

### Export
CSV or JSON export of up to 10,000 rows. Respects active filters.

### Geo Backfill
A backfill endpoint resolves geo coordinates for historic events that were ingested without location data — useful when the geo lookup chain was temporarily unavailable.

### Threat Feed Correlation
Incoming IPs are checked against Spamhaus DROP and CINS Army known-bad-IP lists. Matches are flagged in the event record and surfaced in the Intel view.

---

## Dashboard Views

| View | Description |
|------|-------------|
| **Map** | Three switchable map modes (command canvas / Leaflet / Cobe globe) |
| **Log** | Filterable real-time event table with anomaly score column |
| **Intel** | Threat intelligence summary + threat feed correlation results |
| **Stats** | Attack heatmap, time-series chart, per-route and per-reason breakdowns |
| **Blocklist** | Active IP and country blocks; add/remove/expire from this view |
| **Rules** | Alert rule CRUD; brute-force and anomaly score thresholds |
| **Replay** | Scrub through a time window and replay attack patterns frame-by-frame |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend runtime | Cloudflare Workers (edge, zero cold starts) |
| Database | D1 — SQLite at the edge |
| Backend framework | Hono (TypeScript) |
| Frontend framework | Next.js 15 (App Router) |
| Map — interactive | Leaflet + OpenStreetMap tiles |
| Map — 3D globe | Cobe (WebGL) |
| Charts | recharts |
| Animation | framer-motion |
| Streaming | Server-Sent Events (SSE) |
| IP enrichment | AbuseIPDB |
| Threat feeds | Spamhaus DROP, CINS Army |
| Language | TypeScript (end-to-end) |

---

## Key Engineering Decisions

**Edge-native — no separate server layer.** Every operation (ingestion, SSE streaming, block enforcement, geo lookup) runs inside a Cloudflare Worker. There is no Node.js server, no separate database host, and no proxy layer to maintain.

**SSE over WebSockets.** The dashboard's real-time feed uses Server-Sent Events rather than WebSockets. Auth-failure data flows in one direction (server → browser); SSE is simpler, uses standard HTTP/2 multiplexing, and avoids the Durable Objects requirement that full-duplex WebSocket connections impose on CF Workers.

**`waitUntil`-deferred enrichment.** Geo resolution, anomaly scoring, alert evaluation, and KV cache writes all run in `waitUntil` so they do not add latency to the ingest response path. The `POST /ingest` endpoint returns `202` immediately; enrichment completes asynchronously within the same Worker invocation.

**Block enforcement as middleware.** IP and geo blocks are enforced by a Hono middleware that reads from D1 on every request — no external firewall, no WAF rule propagation lag. A Workers KV cache sits in front of the D1 read to keep enforcement latency under 1 ms.

**KV deduplication.** The SSE endpoint uses Workers KV to track the last-seen event ID per connection, preventing duplicate delivery when a client reconnects after a brief drop.

**Anomaly scoring at ingestion.** Scoring runs inline at write time (or in `waitUntil` for slower heuristics), so every event in D1 already carries its score. The dashboard never needs to compute scores on the read path.

---

## Security Architecture

- All admin endpoints require a pre-shared bearer token validated via Web Crypto `timingSafeEqual`
- Share tokens are scoped to read-only aggregate data — raw IPs and individual events are never exposed
- Block enforcement middleware sits at the first Hono layer — blocked IPs never reach route handlers
- IP enrichment calls to AbuseIPDB are proxied through the Worker — API keys are never exposed to the browser
- CSRF token required on all state-mutating endpoints
- Auth-fail logging includes a salted hash of the IP (not the raw IP) in the SSE dedup key to prevent KV-based enumeration

---

## License

MIT — see [LICENSE](LICENSE).

Authorized-use requirement applies regardless of license: only monitor systems you own or have explicit written authorization to observe.
