<div align="center">

> **Authorized Use Only** — Attack surface mapping and threat modeling operations must be conducted exclusively against systems you own or have explicit written authorization to assess. Unauthorized use is illegal under the CFAA and equivalent laws globally.

# AttackMap

**Attack surface mapper and threat modeling framework — MCP-compatible, local-first.**

[![Python](https://img.shields.io/badge/Python-3.8%2B-blue.svg)](https://www.python.org/)
[![MCP](https://img.shields.io/badge/MCP-Compatible-purple.svg)](#mcp-integration)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

</div>

## Overview

AttackMap enumerates and scores your attack surface across networks, services, and applications — then exposes findings via MCP so automation clients (Claude Code, Cursor, Copilot) can query and reason about them interactively. All scan results stay on your local machine; no data is sent to external services.

## Architecture

```
Automation Client (Claude Code / GPT / Cursor)
  └── MCP Protocol
        └── AttackMap Server
              └── Attack Surface Engine
                    ├── Network Scanner (open ports, services, OS detection)
                    ├── Service Scanner (version fingerprinting, CVE correlation)
                    └── App Scanner (web app vectors, misconfigs, exposed endpoints)
```

No data leaves your local machine — local-first, zero exfil.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Python 3.8+ |
| Protocol | MCP (Model Context Protocol) |
| API | REST — local server |
| Architecture | Local-first, no cloud dependency |

## How It Works

- **Enumerates** exposed services, open ports, and app-layer attack vectors
- **Scores** each vector by exploitability + exposure level (0–10 scale)
- **Produces** structured threat reports in JSON and human-readable format
- **MCP integration** — automation clients query findings interactively: "What are my highest-risk open ports?" "Which services are running outdated versions?"
- **Zero-trust** — all scan results stay on local machine, no exfiltration

## Key Engineering

- **IP allowlist** — admin operations locked to authorized networks; IPv6 prefix matching requires /64 minimum specificity to prevent broad-prefix bypass
- **Authorized-use-only design** — scan scope validated against explicit target whitelist before execution
- **Structured output** — findings are machine-readable JSON for downstream tooling + LLM analysis
- **MCP server mode** — runs as persistent MCP server; automation clients can query findings across a session

