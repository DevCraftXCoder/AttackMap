# AttackMap — Planned File Structure

This file documents the intended layout once development begins.
All paths are relative to the repo root.

```
attack-map/
|
+-- server.py                   # FastAPI REST API server (127.0.0.1:8900)
+-- mcp_server.py               # MCP server (stdio transport for AI clients)
+-- scope_enforcer.py           # Target allowlist validation (runs before every operation)
+-- requirements.txt            # All Python dependencies, pinned versions
+-- requirements-core.txt       # Minimal install (API server only, no heavy deps)
+-- .env.example                # Environment variable template (no real values)
|
+-- src/
|   +-- __init__.py
|   +-- surface/
|   |   +-- __init__.py
|   |   +-- discovery.py        # Service and port enumeration modules
|   |   +-- scoring.py          # CVSS-aligned risk scoring engine
|   |   +-- aggregator.py       # Multi-source result aggregation
|   |
|   +-- threat/
|   |   +-- __init__.py
|   |   +-- modeler.py          # Attack path construction
|   |   +-- attack_mapper.py    # MITRE ATT&CK technique mapping
|   |   +-- chain_builder.py    # Attack chain discovery
|   |
|   +-- intelligence/
|   |   +-- __init__.py
|   |   +-- collector.py        # Threat intelligence aggregation
|   |   +-- enrichment.py       # IOC and context enrichment
|   |
|   +-- reporting/
|   |   +-- __init__.py
|   |   +-- json_report.py      # JSON report generator
|   |   +-- markdown_report.py  # Markdown report generator
|   |   +-- sarif_report.py     # SARIF report generator (CI/CD integration)
|   |
|   +-- api/
|   |   +-- __init__.py
|   |   +-- routes/
|   |       +-- health.py       # GET /health
|   |       +-- surface.py      # POST /surface/scan, GET /surface/results
|   |       +-- threats.py      # POST /threats/model, GET /threats/paths
|   |       +-- reports.py      # POST /reports/generate, GET /reports/{id}
|   |
|   +-- mcp/
|       +-- __init__.py
|       +-- tools.py            # MCP tool definitions (scan, model, report)
|       +-- resources.py        # MCP resource definitions (results, findings)
|
+-- tests/
|   +-- __init__.py
|   +-- test_scope_enforcer.py  # Scope enforcement unit tests
|   +-- test_scoring.py         # Risk scoring unit tests
|   +-- test_attack_mapper.py   # ATT&CK mapping unit tests
|   +-- test_api.py             # API endpoint integration tests
|
+-- output/                     # Assessment results (gitignored)
+-- README.md
+-- LICENSE
+-- SECURITY.md
+-- .gitignore
+-- structure.md                # This file
```

## Environment Variables (.env.example)

```
# Required
API_KEY=changeme-generate-a-strong-random-key

# Target scope (comma-separated CIDRs or exact IPs)
ALLOWED_TARGETS=192.168.1.0/24

# Server
HOST=127.0.0.1
PORT=8900

# Assessment defaults
DRY_RUN=true
SCAN_TIMEOUT_SECONDS=300

# Optional: IP allowlist for admin operations (comma-separated CIDRs)
ADMIN_IP_ALLOWLIST=192.168.1.0/24
```
