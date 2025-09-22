# Multi-tenant Architecture

This document explains how AI Proxy implements multi-tenant isolation and per-tenant
controls. A *tenant* corresponds to a `Group`, and every request must authenticate
with a token that is scoped to exactly one group. The runtime combines persistent
records, Redis caches, middleware, and request-distribution logic to enforce quotas,
model access, and balance policies per tenant.

## Core domain objects

### Groups

Tenants are represented by the `Group` model. Besides the identifier, it records
status, usage counters, allowed model sets, and rate modifiers. Optional balance
alert settings let operators notify when a tenant's credit is low.【F:core/model/group.go†L24-L40】

`GroupStatusEnabled`, `GroupStatusDisabled`, and `GroupStatusInternal` control
whether requests are served. Internal groups bypass balance checks and most limits.【F:core/model/group.go†L18-L38】【F:core/middleware/distributor.go†L221-L247】

Rate limiting is driven by `RPMRatio` and `TPMRatio`, which scale the base
requests-per-minute and tokens-per-minute of each model. `AvailableSets` determines
which enabled model sets from the global catalog this tenant can consume.【F:core/model/group.go†L24-L36】

Group-specific overrides are stored in `GroupModelConfig`, allowing per-tenant
changes to RPM/TPM caps, retry behaviour, pricing, and whether responses should
always be persisted. Saving any of these overrides invalidates the cached group
configuration to keep future requests consistent.【F:core/model/groupmodel.go†L14-L116】

### Tokens

Clients authenticate with tokens bound to a group. A token record carries its
random 48-character key, display name, subnet and model allow-lists, and running
usage counters. The quota system supports both total and rolling-period limits,
with automatic period resets when the billing window changes.【F:core/model/token.go†L33-L101】

`GetAndValidateToken` is the canonical entry point. It resolves the token from the
cache or database, checks that it is enabled, and enforces both total and periodic
quota ceilings before the request is allowed to continue.【F:core/model/token.go†L431-L475】

### Caches

Redis caches (`TokenCache` and `GroupCache`) mirror group/token attributes to avoid
round-trips on every call. When Redis is unavailable, the code transparently falls
back to the database. Cached objects store the derived fields that middleware uses,
such as allowed model sets or quota bookkeeping state.【F:core/model/cache.go†L61-L200】【F:core/model/cache.go†L308-L346】

Token caches remember the models explicitly granted to a key and resolve additional
models through the group’s enabled sets. The helper methods (`SetAvailableSets`,
`SetModelsBySet`, `ContainsModel`) ensure the router only forwards requests for
permitted models.【F:core/model/cache.go†L82-L137】

Cached groups retain per-model overrides and balance alert thresholds. Update flows
(including group edits, model overrides, or token changes) always clear the relevant
cache entry so the next request reloads fresh configuration.【F:core/model/groupmodel.go†L46-L116】【F:core/model/group.go†L209-L258】

## Authentication flow

All public relay endpoints attach `TokenAuth`. The middleware strips standard
prefixes from the `Authorization`/`X-Api-Key` header, loads and validates the token,
and retrieves the owning group from cache. Disabled tokens or groups immediately
abort the request. For internal keys, the middleware synthesises an internal group
profile that exposes every enabled model set.【F:core/middleware/auth.go†L74-L158】

After authentication, the token cache is enriched with the group’s allowed sets and
the globally enabled models. These values are stored in the request context for
later stages (e.g. distributor, logging, billing).【F:core/middleware/auth.go†L128-L158】

## Request distribution and enforcement

`distribute` is the central middleware that evaluates tenant policy before a request
reaches any channel. It first verifies the tenant has sufficient balance, emitting
throttled alerts when the balance drops below the configured threshold, and blocks
requests if the tenant has no credit.【F:core/middleware/distributor.go†L212-L313】

Next, it extracts the target model and checks both global availability and per-token
permissions via `ContainsModel`. Requests for unknown or disallowed models return a
`404` to the caller.【F:core/middleware/distributor.go†L349-L399】【F:core/model/cache.go†L90-L107】

### Rate limiting

Per-tenant rate enforcement is layered:

1. The base model configuration is adjusted with tenant-specific overrides and the
   `RPMRatio`/`TPMRatio`. A usage-dependent multiplier (`calculateGroupConsumeLevelRatio`)
   can further reduce throughput as the tenant’s spend grows.【F:core/middleware/distributor.go†L27-L120】
2. Requests feed into a distributed limiter (`reqlimit`), which tracks per-group,
   per-model, and per-token-name counters. When the effective RPM or TPM limit is
   exceeded, the middleware sets `X-RateLimit-*` headers and aborts with `429`.【F:core/middleware/distributor.go†L122-L188】
3. Rejections are still recorded through the asynchronous consumption pipeline so
   monitoring and billing systems see the attempted usage.【F:core/middleware/distributor.go†L441-L458】

### Usage accounting

Successful calls update rolling counters asynchronously. `UpdateGroupUsedAmountAndRequestCount`
increments the aggregate spend for the tenant while also updating Redis so cached
snapshots stay monotonic. Similar helpers exist for tokens, ensuring both levels of
quota enforcement have accurate telemetry.【F:core/model/group.go†L261-L285】【F:core/model/cache.go†L216-L235】

## Administration APIs

Management endpoints are grouped under `/api` and secured with `AdminAuth`. They let
operators create groups and tokens, tune per-tenant limits, assign model overrides,
and inspect usage dashboards. Endpoints for `/api/group/...`, `/api/groups/...`,
`/api/token/...`, and `/api/tokens/...` provide CRUD operations and bulk actions,
forming the operational surface for multi-tenant lifecycle management.【F:core/router/api.go†L21-L153】

Dashboards (`/api/dashboard`, `/api/dashboardv2`) and log search endpoints expose the
metrics collected by the limiter and consumption pipeline, giving visibility into how
tenant policies are performing in production.【F:core/router/api.go†L36-L170】

## Summary

The multi-tenant design combines:

- Persistent group/token models with quotas and overrides.
- Redis caches for low-latency lookups and monotonic usage tracking.
- Authentication middleware that enforces token validity and group status.
- A distributor that checks balance, rate limits, and model permissions before any
  upstream call is attempted.
- Rich administrative APIs for day-two operations.

Together these components provide strong tenant isolation while still allowing fine-
grained customisation for each customer.
