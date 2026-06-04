---
name: blazor_server
description: Security testing playbook for ASP.NET Core Blazor Server applications covering SignalR circuits, component authorization, and state isolation
---

# Blazor Server

Security testing for ASP.NET Core Blazor Server applications. Focus on
circuit-bound state, SignalR message handling, component-level authorization,
and server-side trust boundaries behind interactive UI events.

## Attack Surface

**Core Runtime**
- SignalR endpoint and negotiate flow (`/_blazor`, `/_blazor/negotiate`)
- Circuit lifecycle: connect, reconnect, resume, disposal
- Component rendering pipeline, event dispatch, server-side state updates
- Middleware order: authentication, authorization, antiforgery, CORS, static files

**Authorization Model**
- `[Authorize]` on pages/components/endpoints
- `AuthorizeView` (UI gating) vs server-side enforcement
- Policy/role requirements in components, APIs, hubs, and minimal APIs
- Identity transitions during reconnect and session refresh

**State and Data Handling**
- Circuit-scoped services and accidental singleton/shared state
- `ProtectedLocalStorage` and `ProtectedSessionStorage` assumptions
- Form binding and server event handlers (`EditForm`, event callbacks)
- File upload/download handlers and generated links

**Hybrid Channels**
- Blazor UI event channel (SignalR) plus REST/minimal API endpoints
- Background jobs and queue handlers that act on user-owned resources
- Optional third-party real-time integrations and hubs

**Deployment**
- Reverse proxies/CDN, sticky sessions, TLS termination, websocket upgrades
- Header trust (`X-Forwarded-*`, host handling) and cache boundaries

## High-Value Targets

- SignalR negotiate and websocket upgrade endpoints (`/_blazor*`)
- Auth-sensitive components (billing, role assignment, admin operations)
- Operations hidden only by UI controls (`AuthorizeView`, disabled buttons)
- Circuit reconnection paths and token/session renewal boundaries
- File import/export, report generation, and bulk actions
- Admin/diagnostic routes (`/admin`, `/health`, `/metrics`, debug toggles)
- Linked APIs called by components (often assumed internal/trusted)
- Multi-tenant selectors (organization, workspace, account, project IDs)

## Reconnaissance

**Endpoint and Runtime Discovery**
```
GET /
GET /_blazor/negotiate
GET /_framework/blazor.server.js
GET /_framework/*
GET /swagger
GET /openapi.json
```

Map where business actions execute:
- component event path over SignalR
- backing API/minimal API endpoints
- background task path

**Authorization Mapping**

For each sensitive feature, identify:
- page/component `[Authorize]` requirements
- server handler/API authorization checks
- whether UI visibility differs from server enforcement

**State Boundary Mapping**

For each user action, determine:
- which IDs/tenant values come from client-controlled inputs
- whether ownership/tenant checks are re-evaluated server-side
- whether reconnect changes effective permissions

## Key Vulnerabilities

### Authentication and Authorization

**UI-Only Protection**
- Sensitive actions hidden in UI but still executable server-side
- `AuthorizeView` used as primary protection without server re-checks
- Component parameter tampering reaching privileged handlers

**Reconnect Authorization Drift**
- Circuit reconnect inherits stale privileges after role/session changes
- Permission downgrade not enforced until full re-login
- Mixed auth states across tabs or concurrent sessions

**Cross-Channel Enforcement Gaps**
- Action blocked in API but allowed via component event path (or inverse)
- Different tenant/role checks across UI channel and REST endpoints

### Access Control and Multi-Tenancy

**IDOR in Event Handlers**
- Resource IDs from client events not bound to caller ownership
- Tenant/workspace selectors trusted without membership checks
- Bulk operations applying to cross-tenant resources

**Circuit State Confusion**
- Shared mutable state in singleton/static stores leaks across users
- Reused cached data or component models exposed to other circuits

### Input and Event Integrity

**Event Sequence Abuse**
- Server assumes trusted UI flow order (open -> edit -> save)
- Out-of-order actions bypass validations or approval gates
- Duplicate/replayed events trigger non-idempotent privileged effects

**Model Binding and Validation Gaps**
- Missing server validation on fields normally constrained by UI
- Type coercion and nullable edge cases in bound models
- Over-posting style issues in update handlers

### SignalR and Transport Weaknesses

**Connection and Message Controls**
- Missing per-message authorization checks after handshake
- Weak origin/trust controls around websocket upgrade
- Inadequate limits for connection count, message rate, payload size

**Abuse and Availability**
- Event flood causing CPU/memory pressure in renderer or hubs
- Expensive server callbacks without throttling/back-pressure
- Reconnect storms degrading availability

### Server-Side and Infrastructure Issues

**Header and Host Trust**
- Spoofed forwarded headers influence security decisions
- Host header misuse in generated absolute links

**File and SSRF Paths**
- Upload processing path traversal or unsafe extension assumptions
- Import/webhook URL fetchers vulnerable to SSRF

## Bypass Techniques

- Trigger restricted actions via alternate UI sequence and stale refs
- Replay or race event submissions around reconnect windows
- Swap tenant/resource identifiers through controlled form states
- Cross-channel parity testing: same action via component and API path
- High-frequency event triggering to detect weak throttling or queueing gaps

## Testing Methodology

1. **Enumerate** - Map components, backing APIs, and `/_blazor` lifecycle endpoints.
2. **Action mapping** - For each sensitive feature, pair UI event with server-side action.
3. **Authorization matrix** - Test unauth/user/admin/cross-tenant across component and API channels.
4. **State isolation** - Run multi-session parallel tests and compare outcomes for leakage.
5. **Reconnect/race checks** - Re-test privileged actions during reconnect and auth transitions.
6. **Abuse controls** - Validate rate, payload, and connection limits under burst conditions.

## Validation Requirements

- Side-by-side proof of unauthorized action (owner vs non-owner, tenant A vs B)
- Cross-channel evidence showing inconsistent enforcement (component vs API)
- Reconnect evidence showing privilege drift or stale authorization state
- Event replay/race evidence with deterministic reproduction steps
- Resource impact evidence for flood/abuse findings (latency, errors, dropped sessions)
- Clear mapping of broken trust boundary (client input -> server handler -> business impact)
