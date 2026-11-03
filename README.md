# ASM Shield — Attack Surface Discovery & Vulnerability Assessment

> **Project Status: 7 of 12 scan modules implemented and CLI-integrated**
>
> Domain Intelligence, Subdomain Discovery, IP Intelligence, Port Discovery (with integrated
> Service Identification), Web Application, and TLS & Certificate scans are fully functional.
> DNS Scan is partially built. Scans 9–12 are scaffolded but not yet implemented.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Quick Start](#quick-start)
3. [CLI Usage](#cli-usage)
4. [Scan Modules](#scan-modules)
5. [Folder Structure](#folder-structure)
6. [Technology Stack](#technology-stack)
7. [Development Approach](#development-approach)
8. [Common Result Architecture](#common-result-architecture)
9. [Running Tests](#running-tests)
10. [Authorized Scope & Safe-Scanning Principles](#authorized-scope--safe-scanning-principles)
11. [Development Roadmap](#development-roadmap)

---

## Project Overview

ASM Shield is a CLI-based attack surface discovery and vulnerability assessment tool written in Python. It accepts a domain name or IP address as input and systematically collects and analyses publicly observable information about a target across multiple dimensions: registration data, DNS, subdomains, IP intelligence, open ports and services, web application surface, and TLS/certificate posture.

**Primary goals:**

- Discover and inventory externally visible assets (domains, IPs, services, certificates).
- Identify potential security weaknesses through safe, non-destructive checks.
- Produce structured, evidence-based reports suitable for security review.
- Maintain a modular architecture where each scan type is independent and testable.
- Enable future integration with a web frontend through a FastAPI API layer.

The tool is designed for **authorized security assessments only**. It does not perform exploitation, credential attacks or destructive testing.

---

## Quick Start

```bash
# 1. Enter the backend directory
cd BACKEND

# 2. Create and activate a virtual environment
python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # Linux/macOS

# 3. Install in editable mode with dev dependencies
pip install -e ".[dev]"

# 4. Verify the asm command is available
asm --version

# 5. Launch the interactive menu
asm

# 6. Or run a domain scan directly
asm domain example.com
```

---

## CLI Usage

### Interactive Menu

Run `asm` without arguments to open the interactive numbered menu:

```bash
asm
```

The menu displays all 12 scan modules grouped by category, with live availability status for each. Select a number and press Enter.

### Direct Scan Execution

```bash
# Domain Intelligence scan with terminal report
asm domain example.com

# JSON output (machine-readable, on stdout)
asm domain example.com --output json

# Custom timeout
asm domain example.com --timeout 20

# Help
asm --help
asm domain --help
asm --version
```

### Output Formats

- **Terminal** (default): Styled report with sections per scan type.
- **JSON**: Structured result envelope on stdout. Diagnostic messages go to stderr so the output remains valid JSON.

---

## Scan Modules

| # | Module | Folder | Status |
|---|--------|--------|--------|
| — | Target Resolution | `scan_tools/target_resolution/` | ✅ **Implemented** (shared foundation) |
| 1 | Domain Intelligence | `scan_tools/domain_intelligence/` | ✅ **Implemented** |
| 2 | DNS Scan | `scan_tools/dns_scan/` | 🟡 **Partial** (models + query engine done) |
| 3 | Subdomain Discovery | `scan_tools/subdomain_discovery/` | ✅ **Implemented** |
| 4 | IP Intelligence | `scan_tools/ip_intelligence/` | ✅ **Implemented** |
| 5 | Port Discovery | `scan_tools/port_discovery/` | ✅ **Implemented** |
| 6 | Service Identification | `scan_tools/service_identification/` | ✅ **Integrated** (inside Port Discovery) |
| 7 | Web Application | `scan_tools/web_application/` | ✅ **Implemented** |
| 8 | TLS & Certificate | `scan_tools/tls_certificate/` | ✅ **Implemented** |
| 9 | API Discovery | `scan_tools/api_discovery/` | ⬜ Planned |
| 10 | Cloud & CDN Intelligence | `scan_tools/cloud_cdn_intelligence/` | ⬜ Planned |
| 11 | Email Security | `scan_tools/email_security/` | ⬜ Planned |
| 12 | Vulnerability Assessment | `scan_tools/vulnerability_assessment/` | ⬜ Planned |

---

### Target Resolution (Shared Foundation)

A shared pre-scan module used by multiple scanners to resolve and validate a target before any network activity begins.

**Capabilities:**
- Domain validation and IDNA normalization
- Multi-record DNS resolution (A, AAAA, CNAME) via dnspython
- Result aggregation and deduplication across resolvers
- IP classification (public, private, loopback, link-local, multicast)
- SSRF-safe scope enforcement — private/special-purpose IPs are blocked from external queries
- Structured resolution report with provenance and timing

---

### Domain Intelligence (Scan 1)

Retrieves and interprets publicly available domain registration information via **RDAP (Registration Data Access Protocol)**.

**Capabilities:**
- Domain validation and normalization (IDNA support)
- IANA RDAP bootstrap resolution for authoritative service
- Registrar information (name, IANA ID, abuse contact, registrar URL)
- Registration dates (creation, update, expiry)
- EPP Domain status codes with human-readable explanations (RFC 5731)
- Registration-associated nameservers (distinguished from live DNS delegation)
- Field-level data availability tracking (`AVAILABLE`, `NOT_PROVIDED`, `REDACTED`, `UNSUPPORTED`, `PARSE_ERROR`, `REQUEST_FAILED`)
- Provenance and request execution tracking (server URL, HTTP status, ISO timestamps, monotonic duration)
- Summary counters (Fields Checked, Available, Unavailable, Redacted, Parsing Errors, Request Errors, Warnings)

**Limitations:**
- Only retrieves publicly available RDAP data — privacy-redacted fields remain redacted
- Does not fall back to WHOIS text scraping
- Registration-associated nameservers reflect RDAP registration records, not live DNS queries

---

### DNS Scan (Scan 2) — Partial

Core DNS query engine and data models are implemented. The scanner orchestrator and remaining record-type modules are not yet complete.

**Implemented so far:**
- Structured DNS query engine with UDP + TCP fallback (RFC 7766)
- Empirical query status classification (`RECORD_FOUND`, `NXDOMAIN`, `NODATA`, `TIMEOUT`, `SERVFAIL`, `REFUSED`, `TRANSPORT_ERROR`)
- AA flag evaluation and provenance retention
- Pydantic data models for all planned record types

**Remaining work:** A, AAAA, CNAME, MX, NS, SOA, TXT, DNSSEC record parsers; scanner orchestrator; CLI integration.

---

### Subdomain Discovery (Scan 3)

Multi-technique passive and active subdomain enumeration with configurable technique toggle from the interactive menu.

**Capabilities:**
- **Certificate Transparency** — crt.sh query for issued certificates
- **Passive DNS** — third-party passive DNS sources
- **DNS Brute Force** — SecLists Standard wordlist (20,000 entries) with authorization gate
- **Subdomain Permutation** — generates common permutations of confirmed subdomains
- **Wildcard DNS Detection** — establishes wildcard baseline before validation
- **DNS Resolution & Validation** — validates discovered candidates via live DNS
- **Evidence Correlation** — cross-references candidates across multiple discovery techniques
- **Temporal Tracking** — compares results against prior scan data
- Configurable technique enable/disable from the interactive menu
- Wordlist validation preflight with SHA-256 integrity check
- Authorization confirmation gate required before DNS brute-force

---

### IP Intelligence (Scan 4)

Collects and correlates publicly observable intelligence about one or more IP addresses.

**Capabilities:**
- IP validation and classification (public, private, reserved, loopback, multicast)
- Reverse DNS / PTR lookup
- ASN and BGP prefix lookup
- RDAP registration lookup for IP ranges
- Forward-Confirmed Reverse DNS (FCrDNS) verification
- Approximate geolocation (country, region, city) via public APIs
- Cloud / CDN / hosting provider classification
- Evidence and internal asset correlation
- Graceful partial results — provider errors or timeouts produce `PARTIAL` scans, not crashes

---

### Port Discovery (Scan 5) + Service Identification (Scan 6)

Raw packet-based port scanner using Scapy with integrated service identification.

> **Note:** Service Identification is consolidated inside Port Discovery. Running Port Discovery automatically performs service banner and protocol detection on discovered open ports.

**Capabilities:**
- TCP SYN scanning via Scapy (requires packet capture capability / elevated privileges)
- UDP scanning via Scapy
- Combined TCP + UDP scanning
- Configurable port ranges and protocol selection
- Rate limiting and bounded concurrency to prevent network overload
- Multi-IP scanning with result aggregation and deterministic deduplication
- Scapy capability preflight — blocks scan with a clear error if raw socket access is unavailable
- Service banner grabbing and protocol detection on open TCP ports
- Structured output distinguishing open, closed, and filtered ports

**Dependency:** Requires `scapy >= 2.5` and elevated privileges (root / Administrator) for raw socket access.

---

### Web Application (Scan 7)

Passive web application surface analysis with safe HTTP inspection and bounded crawling.

**Capabilities:**
- Target normalization and DNS resolution with SSRF policy enforcement
- Verified IP pinning — all HTTP requests are bound to the resolved IP, not re-resolved per request
- Initial target probe with full redirect trace inspection
- Bounded same-origin crawl with configurable depth and page limits
- HTTP security header analysis (HSTS, CSP, X-Frame-Options, Referrer-Policy, etc.)
- Cookie attribute analysis (Secure, HttpOnly, SameSite, path scope)
- Public well-known resource checks (`/robots.txt`, `/sitemap.xml`, `/.well-known/security.txt`)
- Link and form extraction from crawled pages
- JavaScript source reference extraction
- Redirect chain analysis (HTTP → HTTPS upgrade, mixed-content detection)
- Finding normalization, deterministic deduplication, and structured report assembly
- No active exploitation or credential attacks performed

---

### TLS & Certificate (Scan 8)

Multi-phase TLS handshake analysis and certificate chain inspection.

**Capabilities:**
- Target normalization, DNS resolution, and scope enforcement
- Append-only `EndpointLedger` for auditable endpoint tracking
- Cryptographically pinned endpoint staging (IP pinning before handshake)
- Controlled baseline TLS handshake capturing empirical TLS version, cipher suite, cipher bits, and ALPN
- Peer leaf certificate binary DER capture without trust assumptions
- Certificate parsing: subject, SANs, issuer, validity period, key type and size, signature algorithm
- Presented certificate chain parsing and trust validation
- Granular TCP connection vs TLS handshake outcome recording
- Cooperative cancellation and reliable socket resource management
- Multi-phase architecture matching the phased implementation reports

---

### API Discovery (Scan 9) — Planned

Planned capabilities: OpenAPI/Swagger detection, endpoint enumeration, auth mechanism identification, HTTP method probing.

---

### Cloud & CDN Intelligence (Scan 10) — Planned

Planned capabilities: CDN provider detection via DNS fingerprinting, cloud hosting provider classification, IP fingerprinting against known cloud ranges, origin server exposure indicators.

---

### Email Security (Scan 11) — Planned

Planned capabilities: SPF record analysis, DKIM key discovery, DMARC policy evaluation, MX record security review.

---

### Vulnerability Assessment (Scan 12) — Planned

Planned capabilities: Cross-scan finding correlation, known CVE mapping for detected software versions, configuration weakness scoring, structured risk reporting.

---

## Folder Structure

```text
BACKEND/
├── main.py                     # Application entry point
├── cli.py                      # CLI interface (Typer + Rich)
├── config.py                   # Configuration from .env
├── pyproject.toml              # Project metadata and asm entry point
├── requirements.txt            # Python dependencies
├── .env / .env.example         # Environment variables
│
├── core/                       # Core orchestration and shared services
│   ├── logger.py               # Centralized logging (stderr)
│   ├── exceptions.py           # Application exception types
│   ├── execution_engine.py     # Scan execution coordination
│   ├── result_manager.py       # ScanResult lifecycle management
│   └── evidence_manager.py     # Evidence collection and storage
│
├── scan_tools/                 # 12 independent scan modules + shared foundation
│   ├── target_resolution/      # Shared pre-scan DNS resolution foundation
│   ├── domain_intelligence/    # Scan 1 — IMPLEMENTED (RDAP)
│   ├── dns_scan/               # Scan 2 — PARTIAL (query engine + models)
│   ├── subdomain_discovery/    # Scan 3 — IMPLEMENTED (8 techniques)
│   │   ├── techniques/         # 8 discovery technique modules
│   │   ├── reporting/          # CLI and JSON report builders
│   │   └── wordlists/          # Wordlist management and validation
│   ├── ip_intelligence/        # Scan 4 — IMPLEMENTED (8 intelligence sources)
│   ├── port_discovery/         # Scan 5 — IMPLEMENTED (Scapy TCP SYN + UDP)
│   │   └── (service_identification integrated here)
│   ├── service_identification/  # Scan 6 — INTEGRATED inside Port Discovery
│   ├── web_application/        # Scan 7 — IMPLEMENTED
│   │   ├── checks/             # Header, cookie, redirect, public resource checks
│   │   └── extractors/         # Link, form, script extractors
│   ├── tls_certificate/        # Scan 8 — IMPLEMENTED (multi-phase)
│   ├── api_discovery/          # Scan 9 — PLANNED (scaffolded)
│   ├── cloud_cdn_intelligence/ # Scan 10 — PLANNED (scaffolded)
│   ├── email_security/         # Scan 11 — PLANNED (scaffolded)
│   └── vulnerability_assessment/ # Scan 12 — PLANNED (scaffolded)
│
├── shared/                     # Reusable cross-module code
│   ├── models/scan_result.py   # ScanResult envelope
│   ├── normalizers/domain.py   # Domain validation
│   ├── constants/              # ScanType, ScanStatus, scan menu definitions
│   └── utilities/              # Dependency checking, common helpers
│
├── api/                        # FastAPI integration (planned)
│   ├── server.py
│   ├── routes/                 # scans, health, capabilities
│   └── schemas.py
│
├── reporting/                  # Report generation (planned)
├── data/                       # Wordlists, fingerprints, etc.
│   └── wordlists/seclists/dns/ # SecLists Standard subdomain wordlist
├── logs/                       # Runtime log output
├── reports/                    # Scan report output
└── venv/                       # Virtual environment (gitignored)
```

---

## Technology Stack

| Purpose | Technology | Status |
|---|---|---|
| Language | Python 3.11+ | ✅ Active |
| CLI | Typer + Rich | ✅ Active |
| HTTP client | httpx | ✅ Active |
| Data validation | Pydantic v2 | ✅ Active |
| Environment | python-dotenv | ✅ Active |
| Testing | pytest | ✅ Active |
| Logging | Python `logging` | ✅ Active |
| DNS queries | dnspython | ✅ Active (Scans 2, 3, 4, Target Resolution) |
| Packet scanning | Scapy | ✅ Active (Scan 5 — Port Discovery) |
| TLS / Certificates | cryptography | ✅ Active (Scan 8 — TLS & Certificate) |
| API (future) | FastAPI + Uvicorn | ⬜ Planned |

---

## Development Approach

This project follows a **CLI-first** development model:

1. **Build the backend first.** All scanning logic lives in `scan_tools/` and `core/`.
2. **Develop one scan module at a time.** Each module is implemented, tested and reviewed independently.
3. **CLI is the primary interface.** The `cli.py` module coordinates scan execution.
4. **API wraps the same core.** The future FastAPI layer will call the same scan managers — no duplicate logic.
5. **Evidence-based results.** Every finding must be backed by observable evidence.
6. **Shared foundations first.** Target Resolution and shared utilities are built before dependent scanners.

---

## Common Result Architecture

All scan modules return a `ScanResult` envelope:

```json
{
  "scan_type": "domain_intelligence",
  "target": "example.com",
  "status": "completed",
  "started_at": "2026-10-01T10:00:00Z",
  "completed_at": "2026-10-01T10:00:03Z",
  "findings": [],
  "errors": [],
  "data": { ... },
  "metadata": {
    "source": "RDAP",
    "retrieved_at": "2026-10-01T10:00:02Z"
  }
}
```

Status values: `completed`, `partial`, `failed`, `unavailable`.

---

## Running Tests

```bash
# Activate virtual environment first, then run from BACKEND/:

# All tests
pytest

# Individual module tests
pytest scan_tools/domain_intelligence/tests/ -v
pytest scan_tools/subdomain_discovery/tests/ -v
pytest scan_tools/ip_intelligence/tests/ -v
pytest scan_tools/port_discovery/tests/ -v
pytest scan_tools/web_application/tests/ -v
pytest scan_tools/tls_certificate/tests/ -v
pytest scan_tools/target_resolution/ -v

# DNS scan (partial — models and query engine tests)
pytest scan_tools/dns_scan/tests/ -v

# CLI tests
pytest tests/test_cli.py -v

# All with verbose output
pytest -v
```

---

## Authorized Scope & Safe-Scanning Principles

1. **Authorized targets only.** Scans run only against explicitly approved targets.
2. **Discovered ≠ authorized.** Subdomains and IPs found during scanning are treated as candidates, not automatically as approved targets.
3. **Non-destructive by default.** No exploitation, credential attacks or intrusive testing.
4. **Rate-limited.** All network operations respect configurable rate limits and timeouts.
5. **Privacy-aware.** RDAP privacy redaction is respected — no bypass attempts.
6. **No stealth or evasion.** The tool identifies itself via User-Agent.
7. **Secrets management.** API keys and credentials are loaded from `.env`, which is excluded from version control.
8. **SSRF protection.** All HTTP-based scanners enforce SSRF-safe policies — private and special-purpose IPs are blocked from external queries.
9. **Brute-force authorization gate.** DNS brute-force requires explicit authorization confirmation at runtime before any queries are sent.

---

## Development Roadmap

| Phase | Description | Status |
|---|---|---|
| 0 | Project scaffolding and documentation | ✅ Complete |
| 1 | Domain Intelligence scan module + CLI | ✅ Complete |
| 1a | Target Resolution shared foundation | ✅ Complete |
| 2 | DNS Scan module | 🟡 In Progress (query engine done) |
| 3 | Subdomain Discovery module | ✅ Complete |
| 4 | IP Intelligence module | ✅ Complete |
| 5 | Port Discovery module (Scapy TCP SYN + UDP) | ✅ Complete |
| 6 | Service Identification (integrated into Port Discovery) | ✅ Complete |
| 7 | Web Application scan module | ✅ Complete |
| 8 | TLS & Certificate module | ✅ Complete |
| 9 | DNS Scan — complete remaining record parsers + orchestrator | ⬜ Not started |
| 10 | API Discovery module | ⬜ Not started |
| 11 | Cloud & CDN Intelligence module | ⬜ Not started |
| 12 | Email Security module | ⬜ Not started |
| 13 | Vulnerability Assessment module | ⬜ Not started |
| 14 | CLI integration and end-to-end testing | ⬜ Not started |
| 15 | Reporting engine | ⬜ Not started |
| 16 | FastAPI integration | ⬜ Not started |
| 17 | Frontend connection | ⬜ Not started |

---

## License

To be determined.
