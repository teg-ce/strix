---
name: messagepack_blazor_server
description: Security testing for MessagePack over SignalR in ASP.NET Core Blazor Server applications
---

# MessagePack for Blazor Server

Security testing for server-side Blazor applications using SignalR with MessagePack payloads. Focus on circuit authorization drift, message integrity, state isolation, and abuse controls under realtime event pressure.

## Attack Surface

**Protocol and Transport**
- SignalR negotiate and upgrade flow (`/_blazor/negotiate`, `/_blazor`)
- WebSocket transport with MessagePack payload framing
- Transport fallback behavior (WebSocket, SSE, long polling)

**Blazor Runtime**
- Circuit lifecycle: connect, reconnect, resume, disposal
- Server-side component event handling and render diffs
- Event ordering assumptions across UI interactions

**Authorization and Identity**
- Authentication at negotiate and connection stages
- Authorization at event handling stage (not only handshake)
- Identity transitions after token refresh, logout, role changes

**State and Data Boundaries**
- Circuit-scoped vs singleton/shared state leakage
- Tenant/resource IDs carried through event payloads
- File operations and generated links triggered by realtime events

**Infrastructure**
- Reverse proxy websocket handling, sticky sessions, reconnect behavior
- Header trust (`X-Forwarded-*`, Host), cache boundaries

## Reconnaissance

**Endpoint Discovery**
```
GET  /
POST /_blazor/negotiate
GET  /_blazor
GET  /_framework/blazor.server.js
GET  /swagger
GET  /openapi.json
```

Map each sensitive feature to:
- the triggering UI action
- resulting realtime message burst
- backing server operation (component handler/API/background task)

**Connection Profiling**

Capture and compare:
- negotiate response metadata
- selected transport and fallback behavior
- auth/session continuity before and after reconnect

**Message Shape Mapping**

For each action, record:
- outbound frame count, type, size, timing
- inbound response burst and sequence
- whether identical actions generate stable message patterns

## Key Vulnerabilities

### Authentication and Authorization

**Handshake-Only Auth**
- Auth validated at connection setup but not on each server-side event
- Reused circuit context allows unauthorized post-connect operations

**Reconnect Privilege Drift**
- Permission changes not reflected until full re-authentication
- Circuit resumes with stale claims after role/session change
- Concurrent tabs/sessions showing inconsistent effective privileges

**UI-Only Enforcement**
- UI visibility controls used as primary protection without server checks
- Hidden/disabled actions still reachable through manipulated event flow

### Access Control and Multi-Tenancy

**Event-Level IDOR**
- Resource or tenant IDs in event-derived inputs not ownership-validated
- Cross-tenant updates from swapped IDs in equivalent action paths

**Cross-Circuit State Leakage**
- Shared mutable state leaking data across circuits/users
- Cached view-model reuse exposing foreign data

### Message Integrity and Sequencing

**Replay and Idempotency Failures**
- Duplicate events re-trigger privileged or financial actions
- No event nonce/sequence validation on sensitive operations

**Out-of-Sequence Execution**
- Server assumes trusted UI state machine ordering
- Invalid sequence (submit/cancel/retry) bypasses business gates

**Schema and Type Confusion**
- Null/missing/empty semantics mishandled
- Numeric boundary and collection-size edge cases bypass validation
- Alternate payload shapes hitting unintended handler branches

### Availability and Abuse

**Message Flood Gaps**
- Missing per-circuit/per-user rate limits
- Large payload handling without hard caps or graceful rejection
- Reconnect storms causing service degradation

**Expensive Callback Abuse**
- High-cost server event handlers without throttling/back-pressure
- Burst-driven queue growth and resource starvation

### Infrastructure Weaknesses

**Proxy and Host Trust**
- Forwarded header spoofing influences authz/routing decisions
- Host poisoning in generated links and callback flows

**Fallback Downgrade Risks**
- Unexpected transport fallback weakens security assumptions
- Inconsistent controls across websocket vs fallback transports

## WAF Evasion

**Encoding and Envelope Variation**
- Binary payloads evade text-centric signatures
- Same business action represented by different field ordering/shape

**Traffic Shaping**
- Low-and-slow event cadence to avoid burst thresholds
- Multi-session spread to bypass per-connection controls

**Channel Blending**
- Blend UI event traffic with normal user actions to hide malicious sequences

## Bypass Techniques

**Sequence Manipulation**
- Trigger equivalent action paths in unusual order to test state machine trust
- Replay valid actions during reconnect windows

**Cross-Channel Parity Testing**
- Compare authorization outcomes for same operation across UI event channel and backing API path

**Identifier Swapping**
- Reuse known-valid action flow while changing tenant/resource context inputs

**Load Pattern Abuse**
- Burst event generation to test weak throttles and payload bounds

## Testing Methodology

1. **Enumerate** - Identify negotiate, connection, and backend operation paths for high-value features.
2. **Baseline** - Record normal message shape and timing per action for a valid user.
3. **Principal matrix** - Test unauth/user/admin and cross-tenant subjects with paired resources.
4. **Parity checks** - Validate equivalent authorization across component events and API endpoints.
5. **Reconnect tests** - Re-run privileged actions through reconnect and auth transition windows.
6. **Replay/sequence tests** - Verify idempotency and illegal transition handling.
7. **Abuse tests** - Validate rate limits, payload limits, and graceful degradation.

## Validation Requirements

- Side-by-side unauthorized action proof (owner vs non-owner, tenant A vs B)
- Reconnect evidence showing stale privileges or authz drift
- Replay/sequence evidence with deterministic reproduction steps
- Cross-channel mismatch evidence (event channel vs API for same action)
- Resource impact evidence for flood conditions (latency, errors, disconnect rate)
- Clear trust-boundary mapping from client input to business impact
