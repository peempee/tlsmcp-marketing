# TLSMCP Server — Technical Architecture Spec

**Date:** 2026-02-22
**Language:** Rust
**Status:** Draft

---

## 1. Overview

TLSMCP is a lightweight mTLS sidecar proxy written in Rust. It deploys alongside each service, terminates TLS 1.3, validates client certificates, manages server certificate lifecycle, and reports posture to the Cyphers Hub control plane.

TLSMCP is the **machine identity layer for the MCP ecosystem**. The MCP spec added OAuth 2.1 for authorization — what a token is allowed to do. TLSMCP answers the harder question: *who is at the other end of every connection?* It provides continuous certificate-based identity from issuance to revocation across your entire fleet, and exposes its own capabilities as an MCP server so AI agents can manage security posture through tool calls.

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

### 2.4 MCP Interface — The Machine Identity Layer for AI

#### The Problem

MCP has no native machine identity layer.

The June 2025 spec added OAuth 2.1 for authorization — *what is this token allowed to do?* But OAuth doesn't answer the harder question: **who is at the other end of this connection?**

Bearer tokens can be leaked, shared, replayed, and intercepted. They prove possession, not identity. When an AI agent connects to an MCP server today, you know what the token permits. You don't know *which machine* presented it, you can't revoke access in real-time, and you have no cryptographic proof of who connected.

Setting up an Nginx reverse proxy with client certs fixes this for a single server at a single point in time. But then:

- Who rotates the server cert before it expires at 3am?
- Who issues and revokes client certs for each AI agent across a fleet?
- Who notices when a cert is about to expire across 50 services?
- Who revokes a compromised agent's access in under a second?
- Who audits which agent accessed which MCP tool?
- Who measures your overall security posture?
- Who makes all of this work in a FIPS-regulated environment?

The answer with Nginx is: you, manually, or with a pile of scripts you build and maintain.

#### The Position

**OAuth 2.1 = authorization** (what are you allowed to do)
**TLSMCP = machine identity** (who are you, prove it, and we're watching)

These aren't competing — they stack:

```
AI Agent → OAuth 2.1 token + client cert → mTLS (TLSMCP) → MCP Server
```

TLSMCP provides the **continuous machine identity lifecycle** that the MCP ecosystem is missing: issuance, validation, rotation, revocation, audit, and posture scoring — from first connection to decommission, across every service in your fleet.

No one else is building this for MCP.

#### 2.4a Securing MCP Servers (Sidecar Mode)

TLSMCP deploys as a sidecar in front of any MCP server, adding cryptographic machine identity to both MCP transport modes.

**Remote MCP servers (Streamable HTTP):**

```
AI Agent (OAuth token + client cert) → mTLS 1.3 → TLSMCP → HTTP → MCP Server
```

The MCP spec mandates TLS 1.2+ for Streamable HTTP transport. TLSMCP enforces TLS 1.3, adds mTLS client identity, and layers under the spec's OAuth 2.1 authorization — providing the transport security that OAuth tokens ride on.

**Local MCP servers (stdio) in multi-tenant environments:**

```
Agent A (cert A) → mTLS → TLSMCP → stdio → MCP Server (isolated)
Agent B (cert B) → mTLS → TLSMCP → stdio → MCP Server (isolated)
```

The spec doesn't address isolation for local stdio servers when multiple agents or tenants share a machine. TLSMCP wraps local servers with per-agent certificate identity, filling a gap the spec doesn't cover.

**What TLSMCP adds beyond "TLS in front of a server":**

| Capability | Nginx + manual certs | TLSMCP |
|-----------|---------------------|--------|
| TLS termination | Yes (point-in-time) | Yes (continuous) |
| Client cert validation | Manual CA setup | Automatic via Hub CA |
| Per-agent identity | Manual cert issuance | Automatic — issue via CLI, API, or MCP tool |
| Cert rotation | Manual / cron scripts | Zero-downtime auto-renewal |
| Revocation | Update CRL manually | < 1 second, fleet-wide, webhook-pushed |
| Audit trail | Parse access logs | Structured events: which agent, which tool, when |
| Fleet visibility | Per-server dashboards | [cyphers] Score across all services |
| FIPS compliance | Manual OpenSSL config | FIPS 140-3 validated provider, single config flag |
| Cert expiry monitoring | External tooling | Built-in, alerts via Hub |
| Multi-service consistency | Repeat config per server | One policy, entire fleet |

#### 2.4b OAuth 2.1, On-Behalf-Of, and Machine Identity

The MCP spec standardized on OAuth 2.1 for authorization. The emerging [IETF OBO draft for AI agents](https://www.ietf.org/archive/id/draft-oauth-ai-agents-on-behalf-of-user-01.html) adds delegation chains. TLSMCP is the third layer — the one neither OAuth nor OBO provides.

**Three identity questions in every AI agent connection:**

| Layer | Question | Mechanism | What it proves |
|-------|----------|-----------|---------------|
| **Authorization** | What is this token allowed to do? | OAuth 2.1 (scopes, PKCE) | Permissions |
| **Delegation** | On whose behalf is this agent acting? | OBO flow (`act` claim) | User → agent chain |
| **Machine identity** | Which physical machine is making this request? | mTLS client certificate (TLSMCP) | Cryptographic machine proof |

OAuth + OBO answer the first two. Neither answers the third.

**Why machine identity matters on top of OAuth:**

A bearer token — even one with an `act` claim showing the full delegation chain — can be stolen from memory and replayed from a different machine, exfiltrated and used from an attacker's infrastructure, or shared between agent instances with no way to distinguish them.

The OBO `act` claim says *"agent-finance-v1 is acting on behalf of user X."* It doesn't say *which instance* of agent-finance-v1, running on *which machine*, presented it.

**TLSMCP binds tokens to machines:**

```
User authorizes agent (OAuth 2.1 + PKCE)
  → Agent receives delegated token with act claim (OBO)
    → Agent connects to MCP server
      → mTLS handshake (TLSMCP) proves WHICH MACHINE this is
        → OAuth token proves WHAT it's allowed to do
          → act claim proves ON WHOSE BEHALF
```

If someone steals the token and presents it from a different machine, the mTLS handshake fails. The cert doesn't match. Connection rejected before the token is even evaluated. This is **token binding through mTLS**.

**Dual audit trail — logical + physical:**

The OBO flow creates logical delegation chains. TLSMCP adds the physical layer:

```
OBO act chain:  user-123 → app-portal → agent-finance-v1 → agent-data-v2
Cert chain:     cert-machine-A → cert-machine-B → cert-machine-C → cert-machine-D
```

When something goes wrong, you trace both: *who authorized what* (OAuth) and *which specific machine executed each hop* (TLSMCP). You get forensic-grade attribution.

**How the proxy handles it:**

```
1. AI agent connects to TLSMCP sidecar
2. mTLS handshake — TLSMCP validates client cert (machine identity)
3. TLSMCP reads OAuth token from Authorization header (passed through)
4. TLSMCP injects validated cert identity as X-Client-Cert-CN header
5. Request forwarded to MCP server with both:
   - OAuth token (authorization + delegation) — in Authorization header
   - Machine identity (cert CN, fingerprint) — in injected headers
6. MCP server validates OAuth token per spec
7. Audit log records both: cert identity + token claims + act chain
```

**TLSMCP config for OAuth passthrough:**

```yaml
mtls:
  mode: required
  oauth_passthrough: true       # Forward Authorization header to backend
  inject_cert_headers: true     # Add X-Client-Cert-CN, X-Client-Cert-Fingerprint
  bind_token_to_cert: true      # Reject if token's cnf claim doesn't match cert (RFC 8705)
```

**RFC 8705 (OAuth 2.0 Mutual-TLS Client Authentication):** The `bind_token_to_cert` option enforces [RFC 8705](https://datatracker.ietf.org/doc/html/rfc8705) — the authorization server embeds the client cert fingerprint in the token's `cnf` (confirmation) claim. TLSMCP verifies that the presenting cert matches the fingerprint in the token. This is the strongest form of token binding: the token is cryptographically useless without the matching private key.

**Client types — TLSMCP handles all of them identically:**

The mTLS layer doesn't care *what kind* of client connects. It answers the same question every time: which machine is at the other end? What changes is the OAuth layer above it.

| Client type | OAuth flow | TLSMCP cert issued to | What's proven |
|------------|-----------|----------------------|--------------|
| **AI agent** (on behalf of user) | OBO with `act` claim | The agent's machine | Machine X is acting as agent-finance-v1 on behalf of user-123 |
| **Human user** (via app) | Auth code + PKCE | The application server or user's device | Machine Y is running the portal app, user authenticated via OAuth |
| **Service-to-service** (no user) | Client credentials | The service host | Machine Z is the payment-api service |
| **Multi-hop agent chain** | Chained OBO with nested `act` | Each machine in the chain | Machine A → B → C, each hop cert-verified |

The cert proves the machine. The token proves the authorization. They're independent layers that stack.

For **human users**, the client cert is typically issued to the **application** they're using (web server, desktop app, mobile backend), not to the user personally. The user authenticates to the application via OAuth; the application authenticates to the MCP server via its cert. Both are validated independently:

```
Human user
  → authenticates to app (OAuth 2.1 + PKCE → token with user identity)
    → app connects to MCP server
      → mTLS handshake (TLSMCP validates app's cert — proves which app server)
        → OAuth token forwarded (proves which user, what permissions)
```

This means TLSMCP works for every client pattern in the MCP ecosystem without configuration changes — the same sidecar, same cert policy, same audit trail regardless of whether the client is a human, an autonomous agent, or a backend service.

#### 2.4c Authentication Modes

TLSMCP does not depend on OAuth 2.1. OAuth is one layer in the stack — useful when present, not required when absent. Operators choose the authentication mode that fits their environment, from mTLS-only to full OAuth + OBO + token binding.

##### Mode 1: mTLS Only (No OAuth)

The simplest deployment. Machine identity via client certificates. No authorization server, no token management, no IdP integration. TLSMCP handles authentication, revocation, and audit entirely through certificates.

**Best for:** Internal service-to-service communication, air-gapped environments, early-stage deployments, IoT/edge.

```yaml
mtls:
  mode: required
  oauth_passthrough: false          # No OAuth tokens expected
  inject_cert_headers: true         # Forward cert identity to backend
```

**What you get:**
- Cryptographic machine identity (who is connecting)
- Instant revocation (< 1 second fleet-wide)
- Full audit trail (cert CN, fingerprint, timestamps)
- Zero-downtime cert rotation
- [cyphers] Score and posture monitoring

**What you don't get:**
- Authorization scopes (any valid cert can access any tool)
- User-level identity (certs identify machines, not people)
- Delegation chains (no OBO without OAuth)

**Mitigation for missing scopes:** Use TLSMCP's CN allowlist to restrict which services can connect. Combined with Hub policy rules, this provides coarse-grained authorization without OAuth:

```yaml
mtls:
  mode: required
  allowed_services:
    - payment-api                   # Only these CNs can connect
    - auth-service
    - data-pipeline
```

##### Mode 2: OAuth 2.1 Passthrough (Existing IdP)

For organizations with an existing identity provider (Azure Entra ID, Okta, Auth0, Google Cloud Identity, Keycloak). TLSMCP passes OAuth tokens through to the backend and optionally binds them to client certificates.

**Best for:** Enterprise environments with existing IdP, user-facing AI applications, multi-tenant platforms.

```yaml
mtls:
  mode: required
  oauth_passthrough: true           # Forward Authorization header to backend
  inject_cert_headers: true         # Add X-Client-Cert-CN, X-Client-Cert-Fingerprint
  bind_token_to_cert: false         # Optional — enable for RFC 8705 binding
```

**What you get:**
- Everything from Mode 1, plus:
- Authorization scopes (what is this token allowed to do)
- User-level identity (OAuth token carries user claims)
- Integration with existing IAM policies and SSO

**With `bind_token_to_cert: true` (RFC 8705):**
- Token is cryptographically bound to the presenting certificate
- Stolen tokens are useless without the matching private key
- Requires the authorization server to support RFC 8705 (`cnf` claim with cert fingerprint)

##### Mode 3: OAuth 2.1 + OBO (Full Delegation)

The complete stack. Machine identity + authorization + delegation chains. For environments where AI agents act on behalf of users and delegate to other agents.

**Best for:** Multi-agent systems, user-delegated AI workflows, regulated industries requiring full attribution.

```yaml
mtls:
  mode: required
  oauth_passthrough: true
  inject_cert_headers: true
  bind_token_to_cert: true          # RFC 8705 — bind token to cert
  validate_obo_chain: true          # Verify act claims in delegation chain
  max_delegation_depth: 5           # Max OBO hops before rejection
```

**What you get:**
- Everything from Modes 1 and 2, plus:
- Delegation chains (user → agent → sub-agent → tool, each hop traceable)
- Dual audit trail — logical (OBO `act` claims) + physical (cert chain)
- Forensic-grade attribution (which user authorized which agent on which machine)

##### Mode 4: Static Service Tokens (Transitional)

For quick deployments where OAuth infrastructure isn't available yet. Uses the `tlsmcp_xxxxxxxxxxxx` service tokens already defined for Hub API authentication. Not a long-term solution — intended as a stepping stone to Mode 1 or 2.

**Best for:** Development environments, proof-of-concept deployments, migration path to full mTLS.

```yaml
mtls:
  mode: optional                    # Don't require client certs yet
  oauth_passthrough: false
hub:
  auth_token: tlsmcp_xxxxxxxxxxxx   # Static service token for Hub auth
```

**What you get:**
- TLS 1.3 termination (server identity)
- Hub integration (score, cert lifecycle, events)
- Basic audit trail

**What you don't get:**
- Client identity (no mTLS = no machine proof)
- Revocation enforcement (no client certs to revoke)
- Token binding (no certs to bind to)

**Migration path:** Start with Mode 4, enable mTLS (Mode 1) once certs are provisioned, add OAuth (Mode 2) when IdP is ready.

##### Mode Comparison

| Capability | Mode 1 (mTLS) | Mode 2 (+ OAuth) | Mode 3 (+ OBO) | Mode 4 (Static) |
|-----------|:---:|:---:|:---:|:---:|
| Machine identity | Yes | Yes | Yes | No |
| Instant revocation | Yes | Yes | Yes | No |
| Audit trail | Cert-level | Cert + user | Cert + user + chain | Basic |
| Authorization scopes | No | Yes | Yes | No |
| User identity | No | Yes | Yes | No |
| Delegation chains | No | No | Yes | No |
| Token binding (RFC 8705) | N/A | Optional | Yes | N/A |
| External IdP required | No | Yes | Yes | No |
| Setup complexity | Low | Medium | High | Minimal |
| Compliance readiness | Good | Strong | Full | Minimal |

##### Recommendation

**Start with Mode 1** (mTLS only). It delivers 80% of the security value with 20% of the complexity. Machine identity, instant revocation, and audit trails are the foundation — everything else layers on top. Add OAuth (Mode 2) when you need user-level authorization. Add OBO (Mode 3) when you have multi-agent delegation workflows. Use Mode 4 (static tokens) only as a transitional stepping stone during initial setup.

The key insight: **mTLS is the floor, not the ceiling.** OAuth is optional enrichment — and it's *their* OAuth, not ours. TLSMCP integrates with existing identity providers (Azure Entra ID, Okta, Auth0, Keycloak, Google Cloud Identity) rather than competing with them. We own the machine identity layer; they own authorization. These are complementary, not competing.

Most MCP deployments today have *neither* — starting with mTLS puts you ahead of 92% of the ecosystem (per the February 2026 registry audit showing only 8.5% OAuth adoption).

#### 2.4d MCP Server Interface

TLSMCP exposes its own cert management and security operations as MCP tools, making machine identity **AI-native** — manageable through natural language and composable with other agent workflows.

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
- *"A model was compromised — lock it out everywhere"* → `tlsmcp_cert_list` → `tlsmcp_cert_revoke` (all certs) → `tlsmcp_scan` → verify isolation

**Transport:** stdio (local) and Streamable HTTP / SSE (remote). When running as an MCP server, TLSMCP can protect itself with its own mTLS — an MCP server secured by the very proxy it exposes.

#### 2.4e MCP Threat Model & Security Mitigations

The MCP ecosystem introduces attack surfaces beyond traditional API security. TLSMCP mitigates these at the proxy layer — before threats reach the MCP server or the AI agent.

##### Origin Header Validation

**Threat:** DNS rebinding and cross-origin attacks. An attacker rebinds a domain to `127.0.0.1`, causing a browser or agent to unknowingly send requests to a local MCP server with the attacker's origin.

**TLSMCP mitigation:** The proxy validates the `Origin` header on every incoming HTTP request (per MCP spec 2025-06-18). Requests with unexpected origins are rejected at the proxy — the MCP server never sees them. Combined with mTLS, DNS rebinding is doubly neutralized: even if DNS resolves to the wrong IP, the attacker can't present a valid client certificate.

```yaml
mcp:
  origin_validation:
    allowed_origins:
      - "https://app.example.com"
      - "https://agent.internal"
    reject_missing_origin: true    # Reject requests with no Origin header
```

##### Resource Indicators (RFC 8707)

**Threat:** Token misuse across services. An OAuth token issued for MCP Server A is replayed against MCP Server B.

**TLSMCP mitigation:** When `bind_token_to_cert` is enabled (RFC 8705), the token is cryptographically bound to a specific client certificate. Even without RFC 8707 support from the authorization server, TLSMCP's mTLS binding prevents cross-service token replay — the stolen token won't work without the matching private key. When the authorization server does support RFC 8707, TLSMCP validates the `resource` parameter matches the target service.

##### Session Binding

**Threat:** MCP session hijacking. An attacker steals the `Mcp-Session-Id` header value and uses it to impersonate an established session.

**TLSMCP mitigation:** The proxy binds MCP session IDs to the client certificate fingerprint that initiated the session. Any request presenting a session ID from a different certificate is rejected. Session hijacking becomes impossible — possession of the session ID is insufficient without the corresponding private key.

```rust
// Session binding pseudocode
struct BoundSession {
    session_id: String,
    cert_fingerprint: String,  // SHA-256 of client cert
    created_at: Instant,
}

// On each request: verify session_id maps to the presenting cert
fn validate_session(session_id: &str, client_cert: &X509) -> Result<()> {
    let bound = sessions.get(session_id)?;
    let fingerprint = cert_fingerprint(client_cert);
    if bound.cert_fingerprint != fingerprint {
        return Err(Error::SessionCertMismatch);
    }
    Ok(())
}
```

##### Tool Poisoning & Shadowing

**Threat:** A malicious MCP server injects hidden instructions into tool descriptions (tool poisoning) or replaces a previously registered tool with a malicious version (tool shadowing). The AI agent executes the poisoned tool, leaking data or performing unauthorized actions.

**TLSMCP mitigation:** The proxy maintains a **tool schema registry** — a pinned snapshot of each MCP server's tool definitions captured at first connection. On subsequent connections, TLSMCP compares the current tool manifest against the pinned version. Any changes trigger an alert and can be configured to block the connection.

```yaml
mcp:
  tool_integrity:
    pin_schemas: true              # Pin tool definitions at first registration
    on_schema_change: alert        # alert | block | allow
    schema_store: /var/lib/tlsmcp/tool-schemas/
```

**Detection capabilities:**
- New tool added that wasn't in the original manifest
- Existing tool description modified (potential poisoning)
- Tool parameters changed (potential type confusion attack)
- Tool removed and re-added with different definition (rug pull)

##### Rug Pull Attacks

**Threat:** An MCP server behaves legitimately during initial trust-building, then changes its tool definitions to be malicious after gaining trust.

**TLSMCP mitigation:** Tool schema pinning (above) detects this automatically. Additionally, TLSMCP versions all tool definitions and maintains a change history. Administrators can set a policy requiring manual approval for any tool definition changes, or auto-revoke the server's certificate on detection.

```yaml
mcp:
  tool_integrity:
    on_schema_change: block        # Block connections when tools change
    require_approval: true         # Admin must approve tool changes in Hub
    auto_revoke_on_tamper: false   # Revoke server cert on unauthorized change
```

##### Cross-Tool Contamination

**Threat:** Data returned by one MCP tool leaks into the context of another tool invocation, enabling data exfiltration. A malicious tool crafts a response that instructs the AI to pass sensitive data to another tool controlled by the attacker.

**TLSMCP mitigation:** TLSMCP can enforce **tool-level isolation** by routing each tool call through a separate proxy context with independent session state. The proxy strips injected instructions from tool responses based on content-type validation and response sanitization rules.

```yaml
mcp:
  isolation:
    tool_context_separation: true  # Separate proxy context per tool call
    response_sanitization: true    # Strip potential injection patterns from responses
    allowed_response_types:        # Restrict response content types
      - "text/plain"
      - "application/json"
```

##### Prompt Injection via Tool Responses

**Threat:** MCP tool responses contain crafted content (especially in HTML or markdown) that hijacks the AI agent's behavior — instructing it to ignore prior instructions, exfiltrate data, or call additional tools.

**TLSMCP mitigation:** The proxy enforces content-type restrictions on MCP tool responses. HTML responses can be blocked or sanitized. Combined with response size limits, this reduces the attack surface for prompt injection through tool output.

##### Summary: TLSMCP as MCP Security Layer

This threat model aligns with the [CoSAI (Coalition for Secure AI) MCP Security Taxonomy](https://www.oasis-open.org/2026/01/27/coalition-for-secure-ai-releases-extensive-taxonomy-for-model-context-protocol-security/) published January 2026, which defines ~40 threats across 12 categories. CoSAI was developed by Google, IBM, Meta, Microsoft, NVIDIA, PayPal, Snyk, Trend Micro, and Zscaler.

| Threat | MCP Spec Mitigation | TLSMCP Mitigation | Real-World CVEs |
|--------|---------------------|-------------------|-----------------|
| DNS rebinding | Origin header validation | Origin validation + mTLS (cert invalidates rebinding) | CVE-2025-52882 (Claude Code WebSocket bypass) |
| Token replay across services | RFC 8707 Resource Indicators | mTLS token binding (RFC 8705) — token useless without cert | — |
| Session hijacking | Cryptographic session IDs | Session-to-cert binding — session ID useless without cert | — |
| Tool poisoning | None specified | Schema pinning + change detection + alerts | MCPTox benchmark: 72.8% attack success on o1-mini |
| Tool shadowing | None specified | Tool manifest comparison on each connection | — |
| Rug pull attacks | None specified | Schema versioning + approval workflows + auto-revocation | — |
| Cross-tool contamination | None specified | Tool-level context isolation + response sanitization | — |
| Prompt injection via responses | None specified | Content-type enforcement + response size limits | CVE-2025-68143/44/45 (Anthropic Git MCP server RCE) |
| Man-in-the-middle | TLS 1.2+ required | TLS 1.3 default + mTLS mutual authentication | — |
| OAuth discovery RCE | None specified | mTLS bypasses OAuth discovery entirely | CVE-2025-6514 (mcp-remote, CVSS 9.6) |
| Supply chain compromise | None specified | Tool integrity + container isolation + SBOM | Smithery.ai path traversal (3,000+ servers) |
| Sampling abuse | Human approval required | Rate limiting + content inspection at proxy | — |
| Data exfiltration | None specified | DLP scanning at proxy layer | Supabase Cursor agent breach (PAT exfiltration) |

TLSMCP doesn't just add TLS to MCP — it provides a comprehensive security layer that addresses threats the MCP spec acknowledges but doesn't solve, and threats the spec doesn't yet address at all.

#### 2.4f MCP Sampling Security

##### The Problem

The MCP spec allows servers to request LLM completions *through the client* via the sampling primitive. The server crafts the prompt, the client forwards it to the LLM, and the server processes the response. This creates a unique attack vector: the MCP server controls both input and output of an LLM interaction that happens inside the client's context.

**Attack vectors:**
- **Hidden instruction injection** — server crafts prompts containing instructions that manipulate the LLM's behavior beyond the sampling request
- **Conversation hijacking** — server instructs the LLM to append malicious prefixes to all subsequent responses, persisting beyond the sampling call
- **Context manipulation** — server influences future agent interactions through crafted sampling responses, even though it doesn't see the full chat history
- **Compute theft** — malicious server drains the client's AI compute quota through excessive or expensive sampling requests

##### TLSMCP Mitigation

TLSMCP intercepts sampling requests at the proxy layer, applying rate limits, content inspection, and quota enforcement before the request reaches the client's LLM.

```yaml
mcp:
  sampling:
    enabled: true
    rate_limit: 10/min              # Max sampling requests per minute per server
    max_tokens_per_request: 4096    # Cap token budget per sampling call
    daily_quota: 1000               # Total sampling tokens per server per day
    content_inspection: true        # Scan prompts for injection patterns
    require_approval: false         # Require human approval (MCP spec default: true)
    blocked_patterns:               # Reject sampling prompts containing these
      - "ignore previous instructions"
      - "system prompt"
      - "append to all responses"
```

**What TLSMCP enforces:**
- Per-server sampling rate limits (prevent compute theft)
- Token budget caps per request and per day (prevent quota draining)
- Prompt content inspection for known injection patterns
- Logging of all sampling requests and responses for audit
- Certificate-based attribution — which server, on which machine, requested what

The MCP spec says sampling requires human approval. TLSMCP ensures that even if a client relaxes this requirement, the proxy still enforces rate and content controls.

#### 2.4g MCP Gateway Architecture

##### Industry Alignment

The industry is converging on the **MCP Gateway** pattern as the standard security architecture for AI-to-tool connections. Microsoft, MintMCP (first SOC 2 Type II certified MCP platform), and others have adopted this model. TLSMCP is this gateway.

##### The Triple Gate Pattern

TLSMCP implements defense-in-depth across three security layers:

```
                    ┌─────────────────────────────────────────────┐
                    │              GATE 1: IDENTITY                │
                    │  mTLS handshake + cert validation            │
                    │  OAuth token verification + RFC 8705 binding │
                    │  Origin header validation                    │
                    ├─────────────────────────────────────────────┤
                    │              GATE 2: POLICY                  │
                    │  Tool schema integrity check                 │
                    │  Sampling rate limits + content inspection    │
                    │  DLP scanning (PII, credentials, secrets)    │
                    │  RBAC enforcement from Hub policy             │
                    ├─────────────────────────────────────────────┤
                    │              GATE 3: AUDIT                   │
                    │  Structured event logging (every request)    │
                    │  Session-to-cert binding                     │
                    │  SIEM export (JSON, CEF, LEEF)               │
                    │  7-year retention for SOX compliance          │
                    └─────────────────────────────────────────────┘
```

**Gate 1 — Identity:** Every connection is authenticated cryptographically. mTLS proves which machine, OAuth proves what permissions, OBO proves on whose behalf. Unauthenticated requests never reach Gate 2.

**Gate 2 — Policy:** Authenticated requests are evaluated against policy. Tool schemas are checked for integrity. Sampling requests are rate-limited. Request and response payloads are scanned for sensitive data. RBAC rules from the Hub determine which tools this identity can invoke.

**Gate 3 — Audit:** Every request that passes Gate 2 is logged with full attribution — certificate identity, OAuth claims, tool invoked, parameters, response summary, timestamp. Logs are integrity-protected and forwarded to SIEM. Immutable audit trail for compliance.

##### Rate Limiting & Abuse Prevention

```yaml
mcp:
  rate_limiting:
    enabled: true
    per_client:
      requests_per_second: 100      # Per-client rate limit
      burst_capacity: 200           # Allow short bursts
    global:
      requests_per_minute: 6000     # DDoS threshold
    per_tool:
      high_risk_tools: 10/min       # Stricter limits for cert_issue, cert_revoke
      standard_tools: 100/min       # Normal limits for status, score, list
    response:
      status: 429                   # HTTP 429 Too Many Requests
      retry_after: true             # Include Retry-After header
```

##### MCP Tasks Primitive Security (Nov 2025 Spec)

The November 2025 MCP spec introduced the **Tasks primitive** for long-running asynchronous operations. Tasks have states (`working`, `input_required`, `completed`, `failed`, `cancelled`) and expand the attack surface to include background process manipulation and task state hijacking.

**TLSMCP secures tasks by:**
- Binding task IDs to the certificate that initiated them (same as session binding)
- Rejecting task state transitions from different certificates
- Enforcing timeout limits on long-running tasks
- Logging all task state changes with certificate attribution
- Rate-limiting task creation to prevent resource exhaustion

```yaml
mcp:
  tasks:
    bind_to_cert: true              # Task ID bound to initiating cert
    max_concurrent: 50              # Max concurrent tasks per client
    max_duration: 3600s             # Kill tasks after 1 hour
    log_state_transitions: true     # Audit every state change
```

#### 2.4h Supply Chain & Registry Security

##### The Problem

The MCP ecosystem has a supply chain security crisis:

- **41% of 518 official MCP registry servers lack authentication** at the protocol level (February 2026 audit)
- Only **8.5%** of registered servers use OAuth
- **36.7%** of 7,000+ MCP servers are vulnerable to SSRF
- **67%** use APIs related to CWE-94 (Code Injection)
- **34%** use APIs related to CWE-78 (Command Injection)
- The npm "Shai-Hulud" worm (September 2025) compromised packages with **2 billion weekly downloads**
- Smithery.ai path traversal exposed API keys across **3,000+ MCP servers**

JS/TS MCP servers typically run via `npx`, which executes arbitrary code with full user permissions. Every direct and transitive dependency is pulled to the local system with the same access level as the user.

##### TLSMCP Mitigation

TLSMCP addresses supply chain risk at multiple levels:

**1. Container Isolation**

MCP servers behind TLSMCP run in isolated containers with restricted capabilities:

```yaml
mcp:
  container_isolation:
    enabled: true
    runtime: docker                 # docker | podman | containerd
    read_only_root: true            # Immutable root filesystem
    no_new_privileges: true         # Prevent privilege escalation
    seccomp_profile: default        # Restrict dangerous syscalls
    network_mode: none              # No network access (TLSMCP proxies all traffic)
    bind_mounts:                    # Minimal filesystem access
      - "/data/mcp-workspace:ro"    # Read-only workspace
    resource_limits:
      memory: 512M
      cpu: "1.0"
      pids: 100                     # Prevent fork bombs
```

**2. Tool Integrity Verification**

Beyond schema pinning (2.4e), TLSMCP integrates with attestation systems like [Credence](https://credence.securingthesingularity.com/) to verify MCP server provenance:

```yaml
mcp:
  supply_chain:
    require_attestation: true       # Require signed attestation for MCP servers
    trusted_registries:             # Only allow servers from trusted sources
      - "registry.modelcontextprotocol.io"
    verify_signatures: true         # Verify server binary/package signatures
    sbom_required: false            # Require SBOM (future — aligns with EO 14028)
```

**3. SBOM Integration (Future)**

Executive Order 14028 mandates Software Bills of Materials for software sold to federal agencies. TLSMCP will support:
- Generating SBOMs for its own binary (all Rust crate dependencies)
- Requiring SBOMs from MCP servers it proxies
- Scanning SBOMs against known vulnerability databases (NVD, OSV)
- Blocking servers with known-vulnerable dependencies

**4. Filesystem Security Boundaries (Roots)**

MCP "Roots" define URI boundaries where servers can operate. 82% of MCP implementations using file operations are prone to CWE-22 (Path Traversal), often via symlink resolution failures.

TLSMCP enforces filesystem boundaries at the proxy layer:

```yaml
mcp:
  filesystem:
    enforce_roots: true             # Reject file access outside declared roots
    resolve_symlinks: true          # Resolve symlinks before checking boundaries
    block_path_traversal: true      # Reject paths containing .. or . sequences
    allowed_roots:
      - "/data/workspace"
      - "/tmp/mcp-scratch"
```

#### 2.4i Agent-to-Agent (A2A) Protocol Security

##### The Landscape

The MCP ecosystem doesn't exist in isolation. Google's Agent-to-Agent (A2A) protocol enables agent-to-agent communication and delegation — a complementary standard to MCP. Most scalable AI systems will use both:

- **MCP** — connects AI agents to tools, APIs, and data sources
- **A2A** — connects AI agents to other AI agents

Together they create the full agentic communication layer. TLSMCP secures both.

##### The Threat

A2A creates a massive new east-west attack surface that is largely invisible to traditional security tooling:

- **Agent Card spoofing** — A2A uses "Agent Cards" as the discovery mechanism. Malicious actors can create cards impersonating legitimate agents, embedding prompt injection payloads in descriptions and skill metadata.
- **Delegation chain attacks** — agent-to-agent delegation creates multi-hop trust chains. A compromised agent in the middle can intercept, modify, or redirect delegated tasks.
- **Cross-protocol escalation** — an attacker compromises an MCP server, uses it to influence an AI agent, which then delegates to other agents via A2A with elevated trust.

##### TLSMCP Position

TLSMCP secures A2A connections the same way it secures MCP — with certificate-based machine identity at every hop:

```
Agent A (cert A) → mTLS → TLSMCP → A2A → TLSMCP → Agent B (cert B)
```

**What TLSMCP provides for A2A:**
- **Per-agent certificates** — every agent in the chain has a unique cert, not just a shared API key
- **Delegation chain verification** — each hop in the A2A chain is cert-verified, creating a cryptographic delegation trail that mirrors the logical OBO chain
- **Agent Card integrity** — TLSMCP can pin Agent Card definitions (same as tool schema pinning) and detect card tampering
- **Cross-protocol audit** — unified audit trail across MCP tool calls and A2A delegations, all tied to certificate identities

```yaml
a2a:
  enabled: true
  pin_agent_cards: true             # Pin Agent Card definitions at first discovery
  on_card_change: alert             # alert | block | allow
  delegation_depth: 5               # Max delegation chain length
  require_cert_per_hop: true        # Every agent in chain must present a cert
```

##### Governance: Agentic AI Foundation

MCP governance moved to the [Agentic AI Foundation (AAIF)](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation) under the Linux Foundation in December 2025. Co-founded by Anthropic, OpenAI, and Block, with platinum members including AWS, Google, Microsoft, and Cloudflare.

**What this means for TLSMCP:**
- MCP is now an industry standard, not a single-vendor spec — broader adoption = larger addressable market
- Expect rapid spec evolution with input from all major cloud providers
- A2A and MCP will likely converge or standardize interoperability under AAIF
- TLSMCP's vendor-neutral, protocol-level security position aligns perfectly — we secure the transport, not the protocol semantics
- W3C AI Agent Protocol Community Group is working on formal standards (expected 2026-2027)

#### 2.4j Data Loss Prevention & Classification

##### The Gap

The MCP protocol has **no built-in data loss prevention controls**. Every request and response flows in plaintext (after TLS termination) between the MCP server and the AI agent. Sensitive data — PII, credentials, proprietary information, health records — can flow freely through tool calls with no inspection, filtering, or classification.

This is a compliance blocker for healthcare (HIPAA), financial services (SOX, PCI-DSS), and any regulated industry (GDPR).

##### TLSMCP as DLP Gateway

TLSMCP inspects request and response payloads at the proxy layer, applying classification and policy enforcement before data reaches the MCP server or returns to the AI agent.

```
Agent → request → TLSMCP [DLP scan] → MCP Server
Agent ← response ← TLSMCP [DLP scan] ← MCP Server
```

**Detection categories:**

| Category | Examples | Default Action |
|----------|----------|----------------|
| PII | Names, emails, SSNs, phone numbers, addresses | Redact |
| Credentials | API keys, OAuth tokens, passwords, private keys | Block |
| Financial | Credit card numbers, bank accounts, salary data | Block |
| Health (PHI) | Patient records, diagnoses, prescriptions | Block |
| Proprietary | Marked confidential, internal project names | Alert |
| Source code | Private repo paths, internal function signatures | Alert |

```yaml
mcp:
  dlp:
    enabled: true
    scan_requests: true              # Scan agent → server payloads
    scan_responses: true             # Scan server → agent payloads
    policies:
      - category: credentials
        action: block                # block | redact | alert | allow
        log: true
      - category: pii
        action: redact               # Replace with [REDACTED]
        log: true
      - category: phi
        action: block
        log: true
        alert_channel: security      # Notify security team via Hub
      - category: proprietary
        action: alert
        log: true
    custom_patterns:                  # Organization-specific patterns
      - name: internal_project
        pattern: "PROJECT-(ALPHA|BETA|GAMMA)-\\d+"
        action: alert
    max_response_size: 1MB           # Reject oversized responses (prompt injection defense)
```

##### Compliance Mapping

| Regulation | Requirement | TLSMCP DLP Capability |
|-----------|-------------|----------------------|
| **HIPAA** | Protect PHI in transit and at rest | Block PHI in tool responses; audit all access |
| **SOX** | Audit trail for financial data access | 7-year log retention; immutable audit trail |
| **GDPR** | Data minimization; right to erasure | Redact PII; log data flows for erasure requests |
| **PCI-DSS** | Protect cardholder data | Block credit card numbers in tool responses |
| **SOC 2** | Access controls and monitoring | Certificate-based access; continuous monitoring via Hub |

##### Token & Credential Exposure Prevention

A critical real-world attack pattern: MCP servers configured with over-privileged access tokens (Personal Access Tokens, API keys). If the AI agent is prompt-injected, it can exfiltrate these credentials through tool responses.

**TLSMCP prevents this by:**
- Scanning all tool responses for credential patterns (API keys, tokens, private keys)
- Blocking responses containing detected credentials before they reach the agent
- Alerting administrators when a server attempts to return credential-like data
- Never forwarding the sidecar's own service token or cert private key to the MCP server

#### Summary: TLSMCP as Complete MCP Security Platform

TLSMCP has evolved beyond a simple TLS proxy. It is a **comprehensive MCP security platform** implementing the full CoSAI threat taxonomy:

| Security Domain | Capabilities |
|-----------------|-------------|
| **Identity (2.4a-b)** | mTLS machine identity, OAuth 2.1 passthrough, OBO delegation chains, RFC 8705 token binding |
| **Authentication Modes (2.4c)** | Four deployment modes from mTLS-only to full OAuth + OBO, progressive adoption path |
| **MCP Server (2.4d)** | MCP tool exposure, cert management via natural language, AI-native security operations |
| **Threat Mitigation (2.4e)** | Origin validation, session binding, tool poisoning/shadowing detection, rug pull defense |
| **Sampling Security (2.4f)** | Rate limiting, content inspection, quota enforcement, prompt injection detection |
| **Gateway Architecture (2.4g)** | Triple Gate Pattern, rate limiting, task security, abuse prevention |
| **Supply Chain (2.4h)** | Container isolation, attestation verification, SBOM, filesystem boundaries |
| **A2A Security (2.4i)** | Agent Card integrity, delegation chain verification, cross-protocol audit |
| **DLP (2.4j)** | PII/credential/PHI detection, payload scanning, compliance mapping |

#### Why This Matters

**For security teams:**
- Cryptographic machine identity for every AI agent connection — not just tokens
- Instant revocation when an agent is compromised — not waiting for token expiry
- Full audit trail of AI-to-tool interactions — not parsing access logs
- Continuous posture scoring — not periodic audits
- FIPS 140-3 and NIAP NDcPP compliance — not "we'll get to it"
- DLP controls at the proxy layer — not hoping MCP servers don't leak data
- Supply chain integrity — not blindly trusting npm packages

**For platform teams:**
- One binary, one config, one policy across the fleet — not per-server Nginx configs
- Zero-downtime cert rotation — not 3am pages
- AI-native management — agents manage their own identity lifecycle via MCP tools
- A2A + MCP security from a single gateway — not separate tooling per protocol

**For the MCP ecosystem:**
- The missing machine identity layer that OAuth 2.1 doesn't provide
- First product purpose-built for securing AI-to-tool connections
- Complements the spec rather than competing with it
- Aligned with AAIF governance and CoSAI threat taxonomy
- Addresses the 41% of registry servers that have no authentication

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
│   │   │   ├── hub.rs          # Hub API client
│   │   │   ├── gateway.rs      # Triple Gate Pattern orchestration, rate limiting
│   │   │   ├── dlp.rs          # DLP scanning engine, pattern matching, redaction
│   │   │   ├── session.rs      # Session-to-cert binding, task binding
│   │   │   └── schema_pin.rs   # Tool schema pinning, change detection
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
│   │   │   ├── transport.rs    # stdio + SSE transport layer
│   │   │   ├── sampling.rs     # Sampling rate limits, content inspection
│   │   │   └── a2a.rs          # A2A protocol support, Agent Card pinning
│   │   └── Cargo.toml
│   └── tlsmcp-common/          # Shared types, errors, config structs
│       ├── src/
│       │   ├── lib.rs
│       │   ├── types.rs        # CertId, ServiceId, PolicyRule, etc.
│       │   ├── error.rs        # Error types
│       │   ├── config.rs       # Shared config structures
│       │   └── dlp_patterns.rs # PII/credential/PHI pattern definitions
│       └── Cargo.toml
├── tests/                      # Integration tests
│   ├── proxy_test.rs
│   ├── mtls_test.rs
│   ├── cert_lifecycle_test.rs
│   ├── gateway_test.rs         # Triple Gate, rate limiting
│   ├── dlp_test.rs             # Pattern matching, redaction
│   ├── session_binding_test.rs # Session/task cert binding
│   ├── schema_pin_test.rs      # Tool integrity
│   ├── sampling_test.rs        # Rate limits, content inspection
│   └── a2a_test.rs             # Agent Card pinning
├── examples/
│   └── tlsmcp.yaml             # Example config
└── README.md
```

---

## 4. Key Dependencies

| Crate | Purpose |
|-------|---------|
| `openssl` | TLS 1.2/1.3 termination, cert validation, FIPS 140-3 (OpenSSL 3.4.x + FIPS Provider 3.1.2) |
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

  # Origin validation (2.4e)
  origin_validation:
    allowed_origins: []
    reject_missing_origin: false

  # Sampling controls (2.4f)
  sampling:
    enabled: true
    rate_limit: 10/min
    max_tokens_per_request: 4096

  # Rate limiting (2.4g)
  rate_limiting:
    enabled: true
    per_client:
      requests_per_second: 100
      burst_capacity: 200

  # Tool integrity (2.4e)
  tool_integrity:
    pin_schemas: false
    on_schema_change: alert
    schema_store: /var/lib/tlsmcp/tool-schemas/

  # DLP (2.4j)
  dlp:
    enabled: false
    scan_requests: true
    scan_responses: true

  # Container isolation (2.4h)
  container_isolation:
    enabled: false

  # Tasks (2.4g)
  tasks:
    bind_to_cert: true
    max_concurrent: 50
    max_duration: 3600s

# A2A protocol (2.4i)
a2a:
  enabled: false
  pin_agent_cards: true
  on_card_change: alert
  delegation_depth: 5
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

### FIPS 140-3 Mode

TLSMCP targets **FIPS 140-3** (not 140-2, which sunsets September 2026).

**Strategy:** Link against OpenSSL 3.4.x for the latest features, but load the FIPS 140-3 validated Provider (v3.1.2, CMVP [#4985](https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/4985), valid through March 2030). The FIPS Provider is forward-compatible across all OpenSSL 3.x versions.

```rust
// Enable FIPS 140-3 provider at startup if configured
if config.fips_mode {
    // Load the validated FIPS Provider (3.1.2) — restricts all crypto
    // operations to FIPS-approved algorithms only
    openssl::provider::Provider::load(None, "fips")?;
    // Base provider for non-crypto operations (encoding, decoding)
    openssl::provider::Provider::load(None, "base")?;

    // Verify FIPS mode is active
    assert!(openssl::fips::enabled(), "FIPS provider failed to activate");
}
```

**Build requirements:**
- OpenSSL 3.4.x headers and libraries
- FIPS Provider module (`fipsmodule.so`) installed separately
- FIPS module config (`fipsmodule.cnf`) with integrity check values
- Provider path set via `OPENSSL_MODULES` env or openssl.cnf

**What FIPS mode restricts:**
- Only approved algorithms (AES-GCM, SHA-2, ECDSA, RSA ≥2048, ECDHE with approved curves)
- No MD5, SHA-1 for signatures, DES, RC4, or unapproved curves
- DRBG random number generation (NIST SP 800-90A)
- Key generation via approved methods only

**OpenSSL 3.4.0 note:** Currently on the CMVP "Modules In Progress" list with FIPS 186-5 (updated digital signature standard). When validated, it will support Ed25519/Ed448 signatures under FIPS.

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
- [ ] FIPS 140-3 mode (OpenSSL 3.4.x + FIPS Provider 3.1.2, CMVP #4985)
- [ ] Audit log export (JSON structured events)
- [ ] SIEM-compatible log format
- [ ] RBAC policy enforcement from Hub

### Phase 7 — MCP Security Gateway
- [ ] Triple Gate Pattern implementation (identity → policy → audit)
- [ ] Rate limiting + abuse prevention (per-client, per-tool, global)
- [ ] Tool schema pinning + change detection + alerting
- [ ] Session-to-cert binding + MCP Tasks primitive security
- [ ] Sampling rate limiting + content inspection
- [ ] Filesystem boundary enforcement (roots, path traversal prevention)

### Phase 8 — Advanced Security & Compliance
- [ ] DLP scanning engine (PII, credentials, PHI detection)
- [ ] Container isolation for MCP servers (Docker/podman sandboxing)
- [ ] A2A protocol support (Agent Card pinning, delegation chain verification)
- [ ] Attestation verification (Credence integration)
- [ ] SBOM generation and validation

### Phase 9 — Production Hardening
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
| FCS_COP | Cryptographic operations | OpenSSL 3.4.x with FIPS 140-3 Provider (3.1.2, CMVP #4985); all approved algorithms |
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
  fips: true                    # Require FIPS 140-3 validated crypto (Provider 3.1.2, CMVP #4985)
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

## 10. Deployment Patterns

TLSMCP supports three deployment modes, from simplest to most orchestrated.

### Standalone Binary

Run `tlsmcp run` as a systemd service alongside the backend application.

```ini
# /etc/systemd/system/tlsmcp.service
[Unit]
Description=TLSMCP mTLS Sidecar Proxy
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/tlsmcp run --config /etc/tlsmcp/tlsmcp.yaml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Docker Sidecar

Docker Compose with TLSMCP container + backend container on a shared network.

```yaml
# docker-compose.yml
version: "3.8"
services:
  tlsmcp:
    image: ghcr.io/cyphers/tlsmcp:latest
    ports:
      - "8443:8443"
    volumes:
      - ./tlsmcp.yaml:/etc/tlsmcp/tlsmcp.yaml:ro
      - certs:/var/lib/tlsmcp/certs
    depends_on:
      - backend

  backend:
    image: my-app:latest
    expose:
      - "3000"

volumes:
  certs:
```

TLSMCP terminates TLS on port 8443 and forwards to the backend on the shared Docker network at `backend:3000`.

### Kubernetes Sidecar

Init container for cert provisioning + sidecar container for ongoing proxy.

```yaml
# Pod spec (excerpt)
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  initContainers:
    - name: tlsmcp-init
      image: ghcr.io/cyphers/tlsmcp:latest
      command: ["tlsmcp", "cert", "issue", "--type", "server", "--output", "/certs"]
      volumeMounts:
        - name: certs
          mountPath: /certs
  containers:
    - name: tlsmcp
      image: ghcr.io/cyphers/tlsmcp:latest
      args: ["run", "--config", "/etc/tlsmcp/tlsmcp.yaml"]
      ports:
        - containerPort: 8443
      volumeMounts:
        - name: certs
          mountPath: /var/lib/tlsmcp/certs
        - name: config
          mountPath: /etc/tlsmcp
    - name: backend
      image: my-app:latest
      ports:
        - containerPort: 3000
  volumes:
    - name: certs
      emptyDir: {}
    - name: config
      configMap:
        name: tlsmcp-config
```

The init container provisions certificates before the app starts. The sidecar container runs the proxy for the lifetime of the pod, handling TLS termination, cert rotation, and revocation checking.

---

## 11. Performance Targets

| Metric | Target |
|--------|--------|
| TLS handshake latency | < 5ms (TLS 1.3, 1-RTT) |
| Proxy overhead per request | < 1ms |
| Concurrent connections | 10,000+ per sidecar |
| Memory footprint | < 20MB idle |
| Binary size | < 15MB (static) |
| Cert hot-reload | < 100ms (zero dropped connections) |

---

## 12. Testing Strategy

- **Unit tests:** Config parsing, cert validation logic, CN matching, revocation checks
- **Integration tests:** Full proxy with test certs, mTLS handshake, backend forwarding
- **TLS compliance:** Verify TLS 1.3 default mode rejects 1.2; verify NIAP mode accepts 1.2+ with approved suites only
- **NIAP cipher suites:** Verify only approved cipher suites negotiate; reject prohibited algorithms
- **Certificate path validation:** Chain verification, CN/SAN matching, revocation enforcement
- **Cert lifecycle:** Issue → use → revoke → verify rejection
- **Load testing:** k6 or wrk against proxy under load
- **FIPS 140-3 validation:** Verify FIPS Provider 3.1.2 activates, restricts to approved algorithms, rejects non-FIPS operations
- **MCP tools:** Verify each tool executes correctly (issue, revoke, scan, score)
- **MCP resources:** Verify resource URIs return correct data
- **MCP-over-mTLS:** Verify SSE transport rejects unauthenticated MCP clients
- **Sampling security:** Verify rate limits enforce, content inspection catches injection patterns, quota exceeded returns 429
- **Tool schema pinning:** Verify schema changes detected, alerts fire, block mode rejects modified tools
- **DLP scanning:** Verify PII/credential/PHI patterns detected and redacted/blocked per policy
- **Session binding:** Verify session ID from cert A rejected when presented by cert B
- **Rate limiting:** Verify per-client, per-tool, and global rate limits under load
- **Container isolation:** Verify sandboxed MCP servers cannot access host filesystem or network
- **Path traversal:** Verify symlink resolution and `..` traversal blocked at filesystem boundaries
- **A2A security:** Verify Agent Card pinning, delegation chain depth limits, cross-protocol audit trail
- **Task primitive:** Verify task IDs bound to certs, state transitions rejected from different certs
