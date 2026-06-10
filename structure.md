# AttackMap — Technical Overview

AttackMap is a real-time auth-failure geo-dashboard running entirely on Cloudflare's edge. It ingests failed login attempts from an API server, stores them in D1 (Cloudflare's SQLite-at-the-edge database), and renders them as a live geo-visualization with country clustering, a heatmap, and a streaming event feed.

## Data pipeline

Auth failures are written to D1 on every rejected login attempt, carrying the IP address, country code, geo coordinates, route, and failure reason. A Server-Sent Events endpoint streams new rows to connected dashboards in real time. Geo coordinates that arrive without location data are resolved asynchronously via a Zippopotam + U.S. Census fallback chain, with results cached in Workers KV to avoid repeat lookups.

## Dashboard surfaces

The dashboard offers multiple views: a Leaflet map with country-level clustering, a Cobe 3-D globe, a time-series chart, and a tabular list. A heatmap view shows attack frequency by day-of-week and hour. Per-IP deep-probe enrichment pulls AbuseIPDB reputation data and reverse-lookup results on demand. Admins can block individual IPs or entire countries with optional expiry; blocks are written to D1 and enforced by Worker middleware on every request.

## Sharing and export

A share-token endpoint lets admins generate a time-limited public link that exposes aggregate statistics (total event count, top countries, top targeted routes) without revealing raw IPs or individual events. A separate export endpoint delivers up to 10,000 rows as CSV or JSON. Workers AI generates a plain-language threat-intelligence summary from recent failures, cached for five minutes to avoid re-running the model on every page load.

## Stack

Cloudflare Workers · D1 · Hono · TypeScript · Next.js 15 · Leaflet · Cobe · Workers AI · AbuseIPDB · Server-Sent Events
