<div align="center">

> **Authorized Use Only** — Attack surface mapping and threat modeling operations must be conducted exclusively against systems you own or have explicit written authorization to assess. Unauthorized use is illegal under the CFAA and equivalent laws globally.

# AttackMap

### Attack Surface Mapper and Threat Modeling Framework for Authorized Security Assessments

[![Python](https://img.shields.io/badge/Python-3.8%2B-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![MCP](https://img.shields.io/badge/MCP-Compatible-purple.svg)](#mcp-integration)
[![Version](https://img.shields.io/badge/Version-1.0.0-orange.svg)](#)
[![Security](https://img.shields.io/badge/Security-Zero--Trust-red.svg)](#security)

**Attack surface mapper and threat intelligence framework for authorized security assessments. Maps exposed services, models threat paths, and produces structured reports via MCP or REST API.**

[Architecture](#architecture) | [Installation](#installation) | [Running](#running) | [Security](#security)

</div>

---

## Overview

AttackMap is a local-first threat modeling framework that enumerates and scores attack surface exposure across networks, services, and application layers. It integrates with AI clients (Claude, GPT, Cursor) via the Model Context Protocol (MCP), enabling interactive threat analysis and collaborative security assessment workflows.

All operations are scoped, audited, and designed to run exclusively against authorized targets. No scan results leave the local machine.

---

## Architecture

```
AI Client (Claude, GPT, Cursor)
  |
  v  (MCP Protocol)
AttackMap MCP Server
  |  +-- Scope enforcer (authorized targets only)
  |  +-- Attack surface collector
  |  +-- Threat path modeler
  |  +-- Report generator
  |
  v
127.0.0.1:8900 (loopback only -- never network-exposed)
  |
  v
Assessment Engine
  |  +-- Surface discovery modules
  |  +-- Risk scoring (CVSS-aligned)
  |  +-- MITRE ATT&CK mapping
  |  +-- Threat intelligence aggregator
  |
  v
./output/  (local filesystem only -- no external transmission)
```

### Security Controls

| Control | Setting | Purpose |
|---------|---------|---------|
| Port binding | `127.0.0.1:8900` | Never reachable from the network |
| Target scope | Allowlist-enforced | Out-of-scope requests fail before execution |
| Dry-run default | `DRY_RUN=true` | Must explicitly opt into live assessment |
| Output | `./output/` local only | No telemetry, no callbacks |
| Auth | API key required | `X-API-Key` header on all endpoints |
| IP allowlist | Configurable CIDR | Blocks VPN exits and foreign IPs |

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Python 3.8+ |
| API framework | FastAPI |
| MCP integration | MCP framework (stdio transport) |
| Threat modeling | Custom scoring engine (CVSS-aligned) |
| ATT&CK mapping | MITRE ATT&CK STIX data |
| Output formats | JSON, Markdown, SARIF |
| Configuration | `.env` + YAML scope files |

---

## Key Features

- **Attack Surface Discovery** — Enumerate exposed ports, services, endpoints, and credentials across a defined target scope
- **Threat Path Modeling** — Map realistic attack chains from initial access to lateral movement and impact
- **MITRE ATT&CK Mapping** — Tag findings against ATT&CK techniques and sub-techniques automatically
- **Risk Scoring** — CVSS-aligned severity scoring with exploitability and impact modifiers
- **MCP Integration** — Expose all capabilities to AI clients (Claude, GPT, Cursor) via MCP protocol
- **Structured Reporting** — Generate JSON, Markdown, and SARIF reports for integration with CI/CD pipelines
- **Zero Telemetry** — All scan results stay on the local filesystem, no external transmission
- **Scope Enforcement** — Allowlist-based target validation blocks out-of-scope operations before execution
- **Dry-Run Default** — All assessments default to simulation mode; live runs require explicit opt-in

---

## Installation

### Prerequisites

- Python 3.8 or later
- pip

### Steps

```bash
git clone https://github.com/DevCraftXCoder/attack-map.git
cd attack-map
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
# Edit .env: set API_KEY, ALLOWED_TARGETS, and optional settings
```

---

## Running

### REST API server

```bash
python server.py
# Binds to 127.0.0.1:8900
```

### MCP server (AI client integration)

```bash
python mcp_server.py
```

Add to your MCP client config (Claude Desktop, Cursor, etc.):

```json
{
  "mcpServers": {
    "attack-map": {
      "command": "python",
      "args": ["/path/to/attack-map/mcp_server.py"],
      "env": {
        "API_KEY": "your-key-here",
        "ALLOWED_TARGETS": "192.168.1.0/24,10.0.0.0/8"
      }
    }
  }
}
```

### Health check

```bash
curl http://127.0.0.1:8900/health
```

---

## Security

All operations require explicit written authorization from the system owner. AttackMap enforces this at the technical level through scope enforcement, dry-run defaults, and local-only output — but technical controls do not substitute for legal authorization.

See [SECURITY.md](SECURITY.md) for the full security policy, responsible disclosure process, and compliance guidance.

---

## License

MIT License. Copyright 2026 Frxncois. See [LICENSE](LICENSE).
