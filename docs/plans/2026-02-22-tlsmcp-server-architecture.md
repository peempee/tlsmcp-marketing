# TLSMCP Server — Technical Architecture Spec

**Date:** 2026-02-22
**Language:** Rust
**Status:** Draft

---

## 1. Overview

TLSMCP is a lightweight mTLS sidecar proxy written in Rust. It deploys alongside each service, terminates TLS 1.3, validates client certificates, manages server certificate lifecycle, and reports posture to the Cyphers Hub control plane.

TLSMCP also serves a dual role in the MCP (Model Context Protocol) ecosystem: it **secures MCP servers** with mTLS authentication, and it **exposes its own capabilities as an MCP server** so AI agents can manage certificates and security posture through tool calls.

```
                          ┌─────────────────────────────┐
                          │        Cyphers Hub           │
                          │  (Control Plane / CA / API)  │
                          └──────────┬──────────────────┘
                                     │ gRPC / HTTPS
                    ┌────────────────┼────────────────┐
                    ▼                ▼                 ▼
              ┌──────────┐    ┌──────────┐     ┌──────────┐
              │ TLSMCP   │    │ TLSMCP   │     │ TLSMCP   │
              │ Sidecar  │    │ Sidecar  │     │ Sidecar  │
              └────┬─────┘    └────┬─────┘     └────┬─────┘
                   │               │                 │
              ┌────▼─────┐    ┌────▼─────┐     ┌────▼─────┐
              │ Service A │    │ Service B │     │ Service C │
              │ (HTTP)    │    │ (HTTP)    │     │ (HTTP)    │
              └───────────┘    └───────────┘     └───────────┘
```

---

## 2. Components

### 2.1 The Proxy Binary (`tlsmcp`)

The core sidecar. Single statically-linked binary.

**Responsibilities:**
- Listen on a TLS 1.3 port (e.g., `:8443`)
- Terminate TLS, validate client certificates (mTLS)
- Forward decrypted traffic to the local backend over plain HTTP (e.g., `127.0.0.1:3000`)
- Serve as the TLS endpoint for server certificates
- Hot-reload certificates without restart
- Cache revocation lists locally, refresh on interval
- Report connection metrics and cert status to Hub

**Connection flow:**
```
Incoming TLS 1.3 connection
  → OpenSSL handshake
  → Client cert validation (signature, expiry, revocation, CN match)
  → If valid: proxy to backend over HTTP
  → If invalid: terminate connection immediately (no bytes reach app)
```

### 2.2 CLI (`tlsmcp`)

Same binary, different subcommands (like `docker` / `kubectl` pattern).

```
tlsmcp run    --config tlsmcp.yaml       # Start proxy
tlsmcp cert   issue --type client --service payment-api --ttl 24h
tlsmcp cert   list
tlsmcp cert   revoke --id <cert-id>
tlsmcp cert   renew --watch              # Server cert auto-renewal
tlsmcp status                            # Show proxy + cert health
tlsmcp score                             # Fetch [cyphers] Score
```

### 2.3 Cyphers Hub (Control Plane)

Separate service (future build). The Hub is the central brain — every other component talks to it. This section defines the full interface contract.

**Hub provides:**
- Certificate Authority (issue, sign, revoke)
- Certificate policy engine (TTL limits, allowed services, approval rules)
- Revocation distribution (webhook push to sidecars)
- Audit log storage
- [cyphers] Score calculation
- Fleet dashboard data

#### Authentication

All Hub API calls use a **service token** (`tlsmcp_xxxxxxxxxxxx`) passed in the `Authorization` header. Tokens are scoped per service and issued during onboarding.

```
Authorization: Bearer tlsmcp_xxxxxxxxxxxx
```

Future: mutual TLS between sidecar and Hub (the sidecar's own client cert authenticates it to the Hub).

#### Hub REST API

```
# ── Certificate Operations ──
POST   /api/v1/certs/issue          # Issue client or server cert
GET    /api/v1/certs/{id}           # Get cert details
GET    /api/v1/certs?service={id}   # List certs for a service
DELETE /api/v1/certs/{id}           # Revoke cert
PATCH  /api/v1/certs/{id}          # Update cert metadata (e.g., extend TTL)

# ── Server Cert Renewal ──
POST   /api/v1/certs/renew          # Request server cert renewal
GET    /api/v1/certs/renew/{id}     # Check renewal status

# ── Revocation ──
GET    /api/v1/revocations?since={timestamp}  # Delta revocation list
GET    /api/v1/revocations/full               # Full CRL

# ── Policy ──
GET    /api/v1/policy/{service_id}  # Get cert policy for a service
PUT    /api/v1/policy/{service_id}  # Update policy (Hub UI / admin)

# ── Score ──
GET    /api/v1/score                # Overall [cyphers] Score
GET    /api/v1/score/breakdown      # Score per dimension

# ── Events & Metrics ──
POST   /api/v1/events               # Sidecar reports events (batch)
GET    /api/v1/audit?service={id}&from={ts}&to={ts}  # Query audit log

# ── Service Registration ──
POST   /api/v1/services/register    # Register a new sidecar
GET    /api/v1/services/{id}        # Get service details
DELETE /api/v1/services/{id}        # Deregister
```

#### Who Calls What

| Caller | Endpoint | When |
|--------|----------|------|
| **Sidecar → Hub** | `POST /certs/issue` | On startup if no cert cached, or on CLI `cert issue` |
| **Sidecar → Hub** | `GET /revocations?since=` | Every `revocation_refresh` interval (default 5m) |
| **Sidecar → Hub** | `POST /events` | Periodic batch (every 30s) — connection counts, handshake failures, cert validations |
| **Sidecar → Hub** | `POST /certs/renew` | When server cert approaches `renew_before` threshold |
| **Sidecar → Hub** | `GET /policy/{service_id}` | On startup + every `sync_interval` (default 60s) |
| **Sidecar → Hub** | `POST /services/register` | On first startup (registers itself with Hub) |
| **CLI → Hub** | `POST /certs/issue` | `tlsmcp cert issue` |
| **CLI → Hub** | `DELETE /certs/{id}` | `tlsmcp cert revoke` |
| **CLI → Hub** | `GET /certs?service=` | `tlsmcp cert list` |
| **CLI → Hub** | `GET /score` | `tlsmcp score` |
| **MCP → Hub** | All of the above | Via tool handlers — MCP tools delegate to the same Hub API client |
| **Hub → Sidecar** | Webhook push | On cert revocation (immediate, not polled) |

#### Hub → Sidecar Webhooks

The Hub pushes critical events directly to sidecars for real-time response. Sidecars register a webhook endpoint during `POST /services/register`.

```
# Sidecar exposes a local webhook listener (e.g., 127.0.0.1:8444)
# Hub calls this on events that can't wait for the next poll cycle

POST /hooks/revocation
{
  "type": "cert_revoked",
  "cert_id": "c-xxxxx",
  "revoked_at": "2026-02-22T10:30:00Z",
  "reason": "compromised"
}

POST /hooks/policy_update
{
  "type": "policy_changed",
  "service_id": "api-prod-01",
  "changes": ["allowed_services", "max_ttl"]
}

POST /hooks/cert_renewed
{
  "type": "cert_renewed",
  "cert_id": "s-xxxxx",
  "new_cert_id": "s-yyyyy",
  "valid_until": "2027-02-22T00:00:00Z"
}
```

#### Data Flow Diagrams

**Sidecar startup:**
```
Sidecar boots
  → POST /services/register (sends service_id, version, webhook_url)
  → GET /policy/{service_id} (fetch cert policy)
  → POST /certs/issue (get client cert if needed)
  → GET /revocations/full (initial CRL load)
  → Start TLS listener
```

**Ongoing operation:**
```
Every 5m:  GET /revocations?since={last_sync}    → update local CRL cache
Every 30s: POST /events {connections, failures}   → report metrics
Every 60s: GET /policy/{service_id}               → sync policy changes
```

**Cert revocation (real-time):**
```
Admin revokes cert in Hub UI
  → Hub: DELETE /certs/{id}
  → Hub: POST webhook to ALL sidecars → /hooks/revocation
  → Each sidecar: adds cert to local revocation cache immediately
  → Next connection from revoked cert: rejected at handshake
  → Latency: < 1 second from revocation to enforcement
```

**Server cert renewal:**
```
Sidecar detects cert expiry approaching (renew_before threshold)
  → POST /certs/renew (Hub initiates ACME/CA renewal)
  → Hub provisions new cert, stores it
  → Hub: POST /hooks/cert_renewed → sidecar
  → Sidecar: loads new cert, builds new SslAcceptor, atomic swap
  → New connections use new cert; old connections drain on old cert
  → Zero downtime
```

#### Hub Offline / Unreachable

Sidecars must remain functional if the Hub is temporarily unreachable:

- **Cached CRL** — last-known revocation list stays active
- **Cached policy** — last-known policy stays enforced
- **Existing certs** — continue working until expiry
- **Events buffered** — metrics queued locally, flushed when Hub reconnects
- **No new issuance** — `cert issue` fails with clear error until Hub is back
- **Logging** — sidecar logs Hub connectivity status for monitoring

### 2.4 MCP Interface

TLSMCP has a dual relationship with the Model Context Protocol:

#### 2.4a Securing MCP Servers (Sidecar Mode)

TLSMCP deploys as a sidecar in front of any MCP server, adding mTLS authentication to a protocol that currently has none.

```
AI Agent (with client cert) → mTLS 1.3 → TLSMCP Sidecar → stdio/HTTP → MCP Server
```

**The problem this solves:** MCP has no authentication standard. Any client that discovers an MCP endpoint can connect. There is no identity, no revocation, no audit trail for AI-to-tool communication.

**What TLSMCP adds:**
- Every AI agent/client must present a valid client certificate
- Per-agent identity — each agent gets its own cert, fully auditable
- Instant revocation — compromised agent cut off in < 1 second, fleet-wide
- Audit trail — which agent accessed which MCP tool, when
- Zero changes to the MCP server code

#### 2.4b TLSMCP as an MCP Server

TLSMCP exposes its own cert management and security operations as MCP tools, allowing AI agents to manage security posture through natural language.

**MCP Tools:**

| Tool | Description |
|------|-------------|
| `tlsmcp_cert_issue` | Issue a client or server certificate |
| `tlsmcp_cert_revoke` | Revoke a certificate instantly |
| `tlsmcp_cert_list` | List active certs for a service |
| `tlsmcp_cert_status` | Check cert health and expiry |
| `tlsmcp_score` | Get [cyphers] Score |
| `tlsmcp_scan` | Scan a domain's TLS posture |
| `tlsmcp_proxy_status` | Check proxy health and connection stats |

**MCP Resources:**

| Resource | Description |
|----------|-------------|
| `tlsmcp://certs/{service_id}` | Certificate inventory for a service |
| `tlsmcp://score` | Current [cyphers] Score breakdown |
| `tlsmcp://audit/{service_id}` | Recent audit events |
| `tlsmcp://policy/{service_id}` | Active cert policy |

**Example agent workflows:**
- *"Scan api.example.com and tell me what's wrong with its TLS config"* → `tlsmcp_scan` → analysis → `tlsmcp_cert_issue` → `tlsmcp_scan` → verify fix
- *"Revoke all certs for the compromised service"* → `tlsmcp_cert_list` → `tlsmcp_cert_revoke` (for each)
- *"What's our score and how do we improve it?"* → `tlsmcp_score` → actionable recommendations

**Transport:** stdio (local) and SSE over HTTP (remote). When running as an MCP server, TLSMCP can optionally protect itself with its own mTLS — an MCP server secured by the very proxy it exposes.

**Why this matters:**
- **No dashboard required** — agents manage certs through tool calls, not UI clicks
- **Automated incident response** — detect threat → revoke certs → rescan → verify, no human in the loop
- **Continuous posture management** — agent monitors score and acts when it drops
- **Composable** — chains with other MCP tools in an agent's workflow
- **First mover** — no existing solution authenticates and audits AI-to-tool connections

---

## 3. Crate Structure

```
tlsmcp/
├── Cargo.toml                  # Workspace root
├── crates/
│   ├── tlsmcp-proxy/           # Core proxy engine
│   │   ├── src/
│   │   │   ├── main.rs         # Entry point, CLI dispatch
│   │   │   ├── config.rs       # YAML config parsing
│   │   │   ├── proxy.rs        # TLS listener + HTTP forwarding
│   │   │   ├── tls.rs          # OpenSSL TLS 1.3 setup
│   │   │   ├── mtls.rs         # Client cert validation logic
│   │   │   ├── certs.rs        # Cert loading, hot-reload, storage
│   │   │   ├── revocation.rs   # CRL/OCSP cache + refresh
│   │   │   ├── metrics.rs      # Connection stats, reporting
│   │   │   └── hub.rs          # Hub API client
│   │   └── Cargo.toml
│   ├── tlsmcp-cli/             # CLI subcommands (cert, status, score)
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── cert.rs         # cert issue/list/revoke commands
│   │   │   ├── status.rs       # proxy health check
│   │   │   └── score.rs        # fetch + display score
│   │   └── Cargo.toml
│   ├── tlsmcp-mcp/             # MCP server interface
│   │   ├── src/
│   │   │   ├── main.rs         # MCP server entry point
│   │   │   ├── tools.rs        # Tool definitions (issue, revoke, scan, score)
│   │   │   ├── resources.rs    # Resource definitions (certs, score, audit)
│   │   │   ├── handlers.rs     # Tool execution logic
│   │   │   └── transport.rs    # stdio + SSE transport layer
│   │   └── Cargo.toml
│   └── tlsmcp-common/          # Shared types, errors, config structs
│       ├── src/
│       │   ├── lib.rs
│       │   ├── types.rs        # CertId, ServiceId, PolicyRule, etc.
│       │   ├── error.rs        # Error types
│       │   └── config.rs       # Shared config structures
│       └── Cargo.toml
├── tests/                      # Integration tests
│   ├── proxy_test.rs
│   ├── mtls_test.rs
│   └── cert_lifecycle_test.rs
├── examples/
│   └── tlsmcp.yaml             # Example config
└── README.md
```

---

## 4. Key Dependencies

| Crate | Purpose |
|-------|---------|
| `openssl` | TLS 1.3 termination, cert validation, FIPS support |
| `tokio` | Async runtime for concurrent connections |
| `hyper` | HTTP/1.1 + HTTP/2 forwarding to backend |
| `clap` | CLI argument parsing |
| `serde` + `serde_yaml` | Config file parsing |
| `tracing` | Structured logging |
| `reqwest` | HTTPS client for Hub API calls |
| `tonic` | gRPC client (Hub communication, future) |
| `notify` | File watcher for cert hot-reload |
| `dashmap` | Concurrent hashmap for connection/cert state |
| `rmcp` | Rust MCP SDK — tool/resource definitions, stdio + SSE transport |

---

## 5. Configuration

```yaml
# tlsmcp.yaml
version: 1
service_id: api-prod-01

hub:
  url: https://hub.cyphers.ai
  auth_token: tlsmcp_xxxxxxxxxxxx
  sync_interval: 60s            # How often to sync with Hub

listen:
  address: "0.0.0.0:8443"
  protocol: tls13               # tls13 (default) | tls12+ (NIAP mode)

backend:
  address: "127.0.0.1:3000"
  protocol: http                # Plain HTTP to local app

mtls:
  mode: required                # required | optional | disabled
  allowed_services:             # CN allowlist
    - payment-api
    - auth-service
  revocation_refresh: 5m        # How often to refresh CRL

server_cert:
  auto_renew: true
  provider: letsencrypt         # letsencrypt | internal | commercial
  renew_before: 30d             # Renew 30 days before expiry
  domains:
    - api.example.com

client_cert:
  default_ttl: 24h
  max_ttl: 720h                 # 30 days max

logging:
  level: info                   # debug | info | warn | error
  format: json                  # json | pretty

mcp:
  enabled: true
  transport: stdio              # stdio | sse
  sse_address: "127.0.0.1:3001" # Only when transport: sse
  self_secure: false            # Require mTLS for MCP connections (SSE only)
```

---

## 6. TLS Implementation Details

### OpenSSL Binding Strategy

Use the `openssl` crate (Rust bindings to libssl/libcrypto):

```rust
// TLS context setup — supports both TLS 1.3 only and NIAP (TLS 1.2+) modes
use openssl::ssl::{SslAcceptor, SslMethod, SslVerifyMode, SslFiletype, SslVersion};

fn build_tls_acceptor(config: &TlsConfig) -> Result<SslAcceptor> {
    let mut builder = SslAcceptor::mozilla_intermediate_v5(SslMethod::tls_server())?;

    match config.protocol {
        Protocol::Tls13 => {
            // Default: TLS 1.3 only
            builder.set_min_proto_version(Some(SslVersion::TLS1_3))?;
        }
        Protocol::Tls12Plus => {
            // NIAP NDcPP mode: TLS 1.2 + 1.3 with approved cipher suites
            builder.set_min_proto_version(Some(SslVersion::TLS1_2))?;
            builder.set_cipher_list(
                "ECDHE-ECDSA-AES256-GCM-SHA384:\
                 ECDHE-RSA-AES256-GCM-SHA384:\
                 ECDHE-ECDSA-AES128-GCM-SHA256:\
                 ECDHE-RSA-AES128-GCM-SHA256"
            )?;
            builder.set_ciphersuites(
                "TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256"
            )?;
            // Require extended_master_secret (RFC 7627)
            builder.set_options(SslOptions::NO_RENEGOTIATION);
        }
    }

    // Server cert + key
    builder.set_certificate_chain_file(&config.server_cert_path)?;
    builder.set_private_key_file(&config.server_key_path, SslFiletype::PEM)?;

    // mTLS: require client cert
    if config.mtls_mode == MtlsMode::Required {
        builder.set_verify(SslVerifyMode::PEER | SslVerifyMode::FAIL_IF_NO_PEER_CERT);
        builder.set_ca_file(&config.ca_cert_path)?;
    }

    Ok(builder.build())
}
```

### FIPS Mode

OpenSSL 3.x supports FIPS via provider modules:

```rust
// Enable FIPS provider at startup if configured
if config.fips_mode {
    openssl::provider::Provider::load(None, "fips")?;
    openssl::provider::Provider::load(None, "base")?;
}
```

### Certificate Hot-Reload

Watch cert files for changes, rebuild `SslAcceptor` without dropping connections:

```
1. File watcher detects cert change (or Hub pushes new cert)
2. Load + validate new cert
3. Build new SslAcceptor
4. Swap AtomicPtr — new connections use new acceptor
5. Existing connections continue on old acceptor until they close
```

---

## 7. Build Phases

### Phase 1 — Core Proxy (MVP)
- [ ] TLS 1.3 listener with OpenSSL
- [ ] HTTP forwarding to backend
- [ ] Load server cert from file
- [ ] YAML config parsing
- [ ] CLI: `tlsmcp run`
- [ ] Structured logging

### Phase 2 — mTLS
- [ ] Client certificate validation
- [ ] CN allowlist matching
- [ ] Certificate revocation checking (CRL file)
- [ ] Connection rejection for invalid certs

### Phase 3 — Certificate Lifecycle
- [ ] Client cert issuance (local CA for dev, Hub API for prod)
- [ ] Server cert auto-renewal (Let's Encrypt ACME)
- [ ] Certificate hot-reload (zero-downtime rotation)
- [ ] CLI: `tlsmcp cert` subcommands

### Phase 4 — Hub Integration
- [ ] Hub API client (auth, cert sync, event reporting)
- [ ] Revocation list sync from Hub
- [ ] [cyphers] Score fetching
- [ ] CLI: `tlsmcp status`, `tlsmcp score`

### Phase 5 — MCP Interface
- [ ] MCP server with stdio transport
- [ ] Tools: `tlsmcp_cert_issue`, `tlsmcp_cert_revoke`, `tlsmcp_cert_list`, `tlsmcp_cert_status`
- [ ] Tools: `tlsmcp_scan`, `tlsmcp_score`, `tlsmcp_proxy_status`
- [ ] Resources: cert inventory, score breakdown, audit log, policy
- [ ] SSE transport for remote MCP access
- [ ] MCP-over-mTLS: self-secured MCP server mode
- [ ] CLI: `tlsmcp mcp serve` subcommand

### Phase 6 — Enterprise
- [ ] FIPS 140-2 mode (OpenSSL FIPS provider)
- [ ] Audit log export (JSON structured events)
- [ ] SIEM-compatible log format
- [ ] RBAC policy enforcement from Hub

### Phase 7 — Production Hardening
- [ ] Connection pooling + keep-alive to backend
- [ ] Graceful shutdown (drain connections)
- [ ] Health check endpoint
- [ ] Prometheus metrics endpoint
- [ ] Docker image (distroless/static)
- [ ] Kubernetes sidecar manifests

---

## 8. Security Principles

1. **TLS 1.3 by default** — with NIAP-compliant TLS 1.2+ mode for government/regulated deployments
2. **Reject first** — invalid certs terminate the connection before any bytes reach the app
3. **No plaintext outside localhost** — backend traffic is loopback only
4. **Minimal attack surface** — single binary, no plugins, no scripting
5. **Cert pinning optional** — can pin to specific CA or cert fingerprint
6. **Secrets never logged** — private keys and tokens are excluded from all log output
7. **Memory safety** — Rust prevents buffer overflows, use-after-free in proxy logic; OpenSSL handles crypto in C with its own hardening

---

## 9. NIAP Compliance (NDcPP + TLS Functional Package)

TLSMCP targets the [NDcPP v3.0e](https://www.niap-ccevs.org/protectionprofiles/482) (Network Device collaborative Protection Profile) and the [PKG_TLS v2.0](https://www.commoncriteriaportal.org/nfs/ccpfiles/files/ppfiles/PKG_TLS_V2.0.pdf) functional package. This enables listing on the NIAP Product Compliant List (PCL) for US government and defense procurement.

**Note:** NDcPP v4.0 is published and mandatory for new evaluations after June 14, 2026. v4.0 adopts CC:2022 and moves TLS/X.509 into modular functional packages (PKG_TLS v2.x, PKG_X.509 v1.0). Our architecture already aligns with this modular approach — TLS and cert validation are isolated in dedicated modules (`tls.rs`, `mtls.rs`).

### TLS Protocol Requirements

| Requirement | TLSMCP Implementation |
|------------|----------------------|
| TLS 1.2 support (mandatory) | Supported in `tls12+` mode |
| TLS 1.3 support (optional) | Supported — default mode |
| Reject all other SSL/TLS versions | OpenSSL `set_min_proto_version` enforced |
| Abort on version downgrade | Handled by OpenSSL handshake |

### Approved Cipher Suites (TLS 1.2)

Per PKG_TLS v2.0 and RFC 9151 (CNSA Suite Profile), ordered by preference — ECDHE preferred over DHE/RSA, GCM preferred over CBC:

```
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384    (preferred — CNSA)
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
TLS_DHE_RSA_WITH_AES_256_GCM_SHA384
TLS_DHE_RSA_WITH_AES_128_GCM_SHA256
```

### Approved Cipher Suites (TLS 1.3)

```
TLS_AES_256_GCM_SHA384                      (preferred — CNSA)
TLS_AES_128_GCM_SHA256
```

### Prohibited Cryptography

TLSMCP rejects all of the following (enforced via OpenSSL cipher configuration):

- Null encryption
- Anonymous server authentication
- Export-grade algorithms (DES, 3DES, RC2, RC4, IDEA)
- MD5-based hashing
- SHA-1 based cipher suites
- CBC mode suites (allowed by spec but excluded by default — configurable for legacy)

### Key Exchange Requirements

| Algorithm | Parameters | Status |
|-----------|-----------|--------|
| ECDHE | secp256r1, secp384r1, secp521r1 | Supported |
| DHE | ffdhe2048 – ffdhe8192 | Supported |
| RSA key sizes | 2048, 3072, 4096 bits | Supported |

### Signature Algorithms

| Algorithm | Status |
|-----------|--------|
| `ecdsa_secp256r1_sha256` | Supported |
| `ecdsa_secp384r1_sha384` | Supported |
| `rsa_pkcs1_sha256` | Supported |
| `rsa_pkcs1_sha384` | Supported |
| `rsa_pss_rsae_sha256` | Supported |
| `rsa_pss_rsae_sha384` | Supported |

### X.509 Certificate Validation

| Requirement | Implementation |
|------------|----------------|
| X.509v3 chain validation | OpenSSL built-in chain verification |
| CN / SAN matching (DNS, IP, URI) | Custom validation in `mtls.rs` |
| Abort on validation failure | Connection terminated immediately |
| Admin override for invalid certs | Configurable per policy (`cert_override: admin_only`) |
| Wildcard matching rules | No top-level wildcarding (*.com rejected) |

### Revocation Checking

| Method | Status |
|--------|--------|
| CRL (Certificate Revocation List) | Supported — cached locally, synced from Hub |
| OCSP (Online Certificate Status Protocol) | Supported — stapled or live lookup |
| OCSP stapling | Supported — server-side stapling for performance |
| Graceful fallback when revocation unavailable | Configurable: `fail_open` or `fail_closed` (default: `fail_closed`) |

### Session Security

| Requirement | Implementation |
|------------|----------------|
| Secure renegotiation (RFC 5746) | `renegotiation_info` extension enforced |
| `extended_master_secret` (RFC 7627) | Required in TLS 1.2 mode |
| Reject unexpected renegotiation | `SslOptions::NO_RENEGOTIATION` in NIAP mode |
| Session resumption (optional) | Session tickets (RFC 5077), PSK (TLS 1.3) |

### NDcPP v3.0e Security Functional Requirements

Beyond TLS, the NDcPP mandates security functional requirements across six categories. Here's how TLSMCP maps to each:

#### FCS — Cryptographic Services

| SFR | Requirement | TLSMCP Implementation |
|-----|------------|----------------------|
| FCS_CKM | Key generation & lifecycle | OpenSSL key generation; keys stored encrypted at rest |
| FCS_COP | Cryptographic operations | OpenSSL with FIPS provider; all approved algorithms |
| FCS_RBG_EXT | Random bit generation | OpenSSL DRBG (NIST SP 800-90A); entropy documentation required |
| FCS_TLSC_EXT | TLS client | Hub communication uses TLS 1.2+/1.3 client |
| FCS_TLSS_EXT | TLS server | Core proxy TLS listener with approved cipher suites |

#### FAU — Security Audit

| SFR | Requirement | TLSMCP Implementation |
|-----|------------|----------------------|
| FAU_GEN_EXT | Audit data generation | All security-relevant events logged (Table 2 events) |
| FAU_STG_EXT | Protected audit storage | Audit logs integrity-protected; forwarded to Hub/SIEM |

**Auditable events include:**
- Authentication attempts and failures (mTLS handshake accept/reject)
- Certificate operations (issuance, revocation, renewal)
- Configuration changes (policy updates from Hub)
- Administrative actions (CLI commands)
- Proxy startup/shutdown
- Update operations

#### FIA — Identification & Authentication

| SFR | Requirement | TLSMCP Implementation |
|-----|------------|----------------------|
| FIA_UIA_EXT | User identification & auth | Admin auth via Hub credentials; CLI auth via service token |
| FIA_X509_EXT | X.509 certificate validation | Full chain validation, CN/SAN matching, revocation checking |
| FIA_AFL | Auth failure handling | Connection terminated on cert failure; rate limiting on repeated failures |

#### FPT — TSF Protection

| SFR | Requirement | TLSMCP Implementation |
|-----|------------|----------------------|
| FPT_TUD_EXT | Trusted update | Signed binary updates; signature verified before installation |
| FPT_SKP_EXT | TSF data protection | Private keys never logged; config secrets encrypted at rest |
| FPT_TST_EXT | Self-testing | Startup integrity checks on crypto operations; health endpoint |
| FPT_STM_EXT | Trusted timestamps | NTP-synced timestamps on all audit events |
| FPT_APW_EXT | Admin password protection | Service tokens hashed; no plaintext credential storage |

#### FTP — Trusted Path/Channel

| SFR | Requirement | TLSMCP Implementation |
|-----|------------|----------------------|
| FTP_ITC | Trusted channel | All Hub communication over TLS; mutual auth (future) |
| FTP_TRP | Trusted path | Admin CLI communicates with proxy over localhost only |
| FPT_ITT | Internal data transfer | Sidecar ↔ Hub traffic encrypted; webhook payloads signed |

### NIAP Mode Configuration

```yaml
# Enable NIAP-compliant mode
listen:
  protocol: tls12+              # TLS 1.2 + 1.3 with NIAP cipher suites

niap:
  enabled: true
  fips: true                    # Require FIPS 140-2 validated crypto
  revocation_mode: fail_closed  # Reject if revocation status unknown
  extended_master_secret: true  # Enforce RFC 7627
  ocsp_stapling: true           # Enable OCSP stapling
  audit_crypto_ops: true        # Log all cryptographic operations
  secure_update:
    verify_signature: true      # Require signed binary updates
    allowed_signers:            # Trusted update signing keys
      - cyphers-release-key
```

### Certification Path

Building to NDcPP v3.0e / PKG_TLS v2.0 from day one, with an eye on v4.0 (CC:2022) which becomes mandatory June 14, 2026.

**v4.0 key changes that affect us:**
- TLS moves from embedded SFRs to PKG_TLS v2.x (our `tls.rs` module already isolates this)
- X.509 moves to PKG_X.509 v1.0 (our `mtls.rs` module already isolates this)
- Explicit certificate path validation rules (FCO_CPC_EXT.1)
- CC:2022 introduces Parts 4 & 5 with updated evaluation methods

**Formal certification involves:**

1. **Security Target (ST)** — document mapping TLSMCP to NDcPP SFRs (write during Phase 6)
2. **Lab evaluation** — accredited CCTL lab tests against the ST
3. **NIAP validation** — listing on the Product Compliant List

**Entropy documentation** (Appendix D of NDcPP) required for FCS_RBG_EXT — must provide design description, entropy justification, operating conditions, and health testing documentation for our random number generation.

The architecture supports all required SFRs. Formal certification is a business decision based on customer demand, but building compliant from day one makes future certification a documentation exercise, not a re-architecture.

---

## 10. Performance Targets

| Metric | Target |
|--------|--------|
| TLS handshake latency | < 5ms (TLS 1.3, 1-RTT) |
| Proxy overhead per request | < 1ms |
| Concurrent connections | 10,000+ per sidecar |
| Memory footprint | < 20MB idle |
| Binary size | < 15MB (static) |
| Cert hot-reload | < 100ms (zero dropped connections) |

---

## 10. Testing Strategy

- **Unit tests:** Config parsing, cert validation logic, CN matching, revocation checks
- **Integration tests:** Full proxy with test certs, mTLS handshake, backend forwarding
- **TLS compliance:** Verify TLS 1.3 default mode rejects 1.2; verify NIAP mode accepts 1.2+ with approved suites only
- **NIAP cipher suites:** Verify only approved cipher suites negotiate; reject prohibited algorithms
- **Certificate path validation:** Chain verification, CN/SAN matching, revocation enforcement
- **Cert lifecycle:** Issue → use → revoke → verify rejection
- **Load testing:** k6 or wrk against proxy under load
- **FIPS validation:** Verify FIPS provider activates and restricts algorithms
- **MCP tools:** Verify each tool executes correctly (issue, revoke, scan, score)
- **MCP resources:** Verify resource URIs return correct data
- **MCP-over-mTLS:** Verify SSE transport rejects unauthenticated MCP clients
