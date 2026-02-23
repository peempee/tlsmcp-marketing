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

Separate service (future build). For now, define the API contract.

**Hub provides:**
- Certificate Authority (issue, sign, revoke)
- Certificate policy engine (TTL limits, allowed services, approval rules)
- Revocation distribution (webhook push to sidecars)
- Audit log storage
- [cyphers] Score calculation
- Fleet dashboard data

**Hub API (gRPC + REST):**
```
POST   /api/v1/certs/issue          # Issue client or server cert
GET    /api/v1/certs/{id}           # Get cert details
DELETE /api/v1/certs/{id}           # Revoke cert
GET    /api/v1/certs/revocations    # Get revocation list (delta)
POST   /api/v1/certs/renew          # Trigger server cert renewal
GET    /api/v1/score                # Get [cyphers] Score
POST   /api/v1/events               # Sidecar reports events/metrics
GET    /api/v1/policy/{service_id}  # Get cert policy for a service
```

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
  protocol: tls13               # TLS 1.3 only, no fallback

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
// TLS 1.3 context setup (simplified)
use openssl::ssl::{SslAcceptor, SslMethod, SslVerifyMode, SslFiletype};

fn build_tls_acceptor(config: &TlsConfig) -> Result<SslAcceptor> {
    let mut builder = SslAcceptor::mozilla_intermediate_v5(SslMethod::tls_server())?;

    // TLS 1.3 minimum
    builder.set_min_proto_version(Some(SslVersion::TLS1_3))?;

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

1. **TLS 1.3 only** — no negotiation to older protocols
2. **Reject first** — invalid certs terminate the connection before any bytes reach the app
3. **No plaintext outside localhost** — backend traffic is loopback only
4. **Minimal attack surface** — single binary, no plugins, no scripting
5. **Cert pinning optional** — can pin to specific CA or cert fingerprint
6. **Secrets never logged** — private keys and tokens are excluded from all log output
7. **Memory safety** — Rust prevents buffer overflows, use-after-free in proxy logic; OpenSSL handles crypto in C with its own hardening

---

## 9. Performance Targets

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
- **TLS compliance:** Verify TLS 1.3 only (reject 1.2 connections), cipher suite enforcement
- **Cert lifecycle:** Issue → use → revoke → verify rejection
- **Load testing:** k6 or wrk against proxy under load
- **FIPS validation:** Verify FIPS provider activates and restricts algorithms
- **MCP tools:** Verify each tool executes correctly (issue, revoke, scan, score)
- **MCP resources:** Verify resource URIs return correct data
- **MCP-over-mTLS:** Verify SSE transport rejects unauthenticated MCP clients
