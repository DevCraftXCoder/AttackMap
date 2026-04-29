# Security Policy

## Reporting a Vulnerability

Do **not** open a public GitHub issue for security vulnerabilities.

Contact via GitHub: [@DevCraftXCoder](https://github.com/DevCraftXCoder)

Response time: 72 hours for acknowledgment, 7 days for critical issues.

---

## Responsible Use

AttackMap is a **security assessment framework** designed exclusively for:

- Authorized penetration testing engagements with written authorization
- Threat modeling exercises on systems you own or operate
- Security research in isolated lab environments
- Defensive analysis to understand attack surface exposure

**Never use AttackMap against systems you do not own or have explicit written authorization to assess.** Unauthorized use is illegal under the Computer Fraud and Abuse Act (CFAA), the UK Computer Misuse Act, and equivalent laws in most jurisdictions.

---

## Security Architecture

### Scope Enforcement

Target validation is the primary safety gate:

- All assessment targets are validated against an `ALLOWED_TARGETS` allowlist before any operation executes
- CIDR notation is required for network ranges — individual IP wildcards are rejected
- Out-of-scope requests return an error and are logged locally — they never execute
- The allowlist is set at startup via environment variable and cannot be modified at runtime via API

### API Authentication

- All API endpoints require a valid API key via the `X-API-Key` header
- API keys are generated locally and stored only in the local `.env` file
- The API server binds exclusively to `127.0.0.1` — not reachable from the network without explicit port forwarding

### Zero-Trust IP Allowlisting

Admin operations require IP allowlist membership in addition to API key authentication:

- Requests must originate from the configured home network CIDR
- VPN exits, proxies, and foreign IPs are rejected even with a valid API key
- IPv6 prefix matching requires minimum /64 specificity to prevent broad-prefix bypass

### Dry-Run Mode (Default On)

All assessment operations default to dry-run mode:

- Dry-run shows what would be collected or analyzed without executing active probes
- Live assessments require an explicit `dry_run: false` parameter per request
- This prevents accidental active scanning during API exploration or testing

### No Telemetry

- No usage data is sent externally
- No scan results, target data, or findings are transmitted to third parties
- All output is written exclusively to the local `./output/` directory
- No callbacks, beacons, or phone-home behavior of any kind

---

## Dependency Security

- All Python dependencies are pinned to specific versions in `requirements.txt`
- Run `pip audit` periodically to check for vulnerable packages
- Dependency updates are always standalone commits, never bundled with feature work

---

## Known Limitations

- Scope enforcement is a defense-in-depth measure, not a substitute for operator legal authorization
- Technical controls do not replace the requirement for explicit written authorization from system owners
- AI-generated assessment parameters should be reviewed before execution in production environments

---

## Compliance Notes

AttackMap is intended for security professionals operating within legal frameworks. Before any assessment, ensure you have:

- Written authorization from the system owner specifying in-scope and out-of-scope assets
- A defined scope document and agreed rules of engagement (RoE)
- Compliance with applicable data protection regulations (GDPR, HIPAA, etc.) when handling assessment findings
- Appropriate professional liability coverage for commercial engagements
