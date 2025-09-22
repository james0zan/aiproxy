# Request Processing Architecture

This document describes how AI Proxy handles each incoming request, the
middleware and plugins involved, and the mechanisms that keep throughput high
while protecting upstream providers.

## Startup and Background Services

`core/main.go` wires up shared infrastructure before any traffic is accepted.
The process enables pprof profiling, configures outgoing notifications, and
initializes Redis, balance billing integrations, the primary SQL databases, and
in-memory model/channel caches during boot.【F:core/main.go†L48-L108】 Two
long-lived goroutines continuously refresh option flags and model/channel cache
snapshots so routing decisions are always made against fresh data without
re-querying the database on every request.【F:core/main.go†L110-L115】 The HTTP
server is built on Gin with panic recovery, structured logging, request ID
injection, and CORS middleware applied globally.【F:core/main.go†L117-L137】

## Entry Pipeline and Authentication

All OpenAI-compatible endpoints hang off `/v1` and inherit two gatekeeper
middlewares: IP blacklist enforcement and token authentication.【F:core/router/relay.go†L9-L148】
`IPBlock` rejects requests from banned addresses early.【F:core/middleware/ipblack.go†L1-L17】
`TokenAuth` validates access keys, enforces subnet restrictions, loads the group
and token cache entries into the Gin context, and exposes the model catalog the
caller is allowed to use.【F:core/middleware/auth.go†L74-L160】 These caches avoid
per-request database round trips while still respecting the latest
synchronizations.

## Distribution Layer Responsibilities

Each relay handler is wrapped by `middleware.NewDistribute`, which executes a
sequence of safeguards before the request reaches an upstream channel.【F:core/controller/relay.go†L23-L120】
Key steps include:

1. **Service availability & bookkeeping** – The middleware stamps the target
   mode onto the context and aborts if global maintenance is enabled.【F:core/middleware/distributor.go†L349-L354】
2. **Balance checks** – Group balances (including dynamic alerts) are fetched or
   reused, and insufficient credit short-circuits the request to protect the
   platform.【F:core/middleware/distributor.go†L212-L313】
3. **Model resolution** – The middleware parses the model from the body, path,
   or pinned store entry, verifies access rights, and persists the resolved
   configuration on the context.【F:core/middleware/distributor.go†L366-L410】【F:core/middleware/distributor.go†L503-L536】
4. **User metadata extraction** – Downstream logging and billing metadata are
   populated via helper functions so retries share consistent attribution.【F:core/middleware/distributor.go†L415-L439】
5. **Rate limiting** – RPM/TPM caps are dynamically scaled using group-specific
   ratios and historical spend, then enforced through `reqlimit` counters that
   also update monitor metrics and emit 429 consumption events when breached.【F:core/middleware/distributor.go†L27-L188】【F:core/middleware/distributor.go†L441-L457】

Requests that clear these checks advance to the relay layer with a fully formed
`meta.Meta` containing group, token, model configuration, timing, and optional
store identifiers (for long-running jobs or responses).【F:core/middleware/distributor.go†L503-L536】

## Channel Selection and Execution

The relay controller picks an initial channel based on explicit headers, pinned
store mappings, auto-banned channel lists, and observed error rates. It folds in
priority weighting so low-error channels receive more traffic while preserving
randomness for load spreading.【F:core/controller/relay-channel.go†L280-L361】 The
same logic powers web-search channel acquisition for the corresponding plugin
needs.【F:core/controller/relay-channel.go†L364-L384】

Before a request leaves the process, the adaptor is wrapped in a stack of
plugins that provide observability and performance enhancements: group/channel
monitoring, Redis and in-memory response caching, fake streaming for otherwise
blocking upstreams, request patching, adaptive timeouts, on-demand web search
channel resolution, and reasoning-mode segmentation.【F:core/controller/relay-controller.go†L86-L118】
Representative examples include:

- The cache plugin stores identical responses in memory or Redis using pooled
  buffers to reduce allocation overhead and accelerate hot prompts.【F:core/relay/plugin/cache/cache.go†L47-L200】
- The timeout plugin tunes per-mode deadlines (with stream-aware overrides) so
  slow providers are isolated without impacting fast paths.【F:core/relay/plugin/timeout/timeout.go†L27-L111】
- The stream-fake plugin upgrades non-streaming chat completions into simulated
  streams and reconverts them locally, preventing client-side timeouts while the
  upstream works.【F:core/relay/plugin/streamfake/fake.go†L29-L177】
- Monitor plugins track per-channel request counts and integrate with automatic
  banning to shed unhealthy backends quickly.【F:core/relay/plugin/monitor/monitor.go†L34-L192】

During execution the adaptor marshals the upstream request, and the monitor
plugin wraps `DoRequest` to measure latency, update rate counters, and record
errors for the auto-ban heuristics.【F:core/relay/plugin/monitor/monitor.go†L72-L147】

## Retry, Parallelism, and Flow Control

After the first attempt the relay controller decides whether a retry is
worthwhile based on plugin feedback. Retry state tracks the last successful
channel, channels lacking permission, and per-channel error rates so subsequent
attempts skip unhealthy providers and optionally inject backoff when the same
channel must be reused.【F:core/controller/relay-controller.go†L272-L523】 Request
bodies are stored via `GetRequestBodyReusable` so the same payload can be replayed
without re-reading the client stream, enabling parallel plugin stages to reuse
bytes safely.【F:core/common/body.go†L64-L117】【F:core/controller/relay-controller.go†L525-L533】

`consume.AsyncConsume` records usage, cost, and metadata on a separate goroutine
for both successful and throttled calls. This decouples billing I/O from the hot
path while still providing a `Wait` hook for graceful shutdown.【F:core/common/consume/consume.go†L18-L118】 Channel metrics and
error-rate updates leverage shared monitor structures so concurrent requests can
adjust routing decisions without locking the request handler.

## Data Management and Acceleration

Model caches loaded during authentication provide low-latency access to
available models per group without repeated SQL queries.【F:core/middleware/auth.go†L128-L160】 Store records for long-lived
operations (video generations, responses) are fetched once and cached for retry
selection, and the relay persists fresh store metadata after successful calls so
follow-up polling can be routed back to the same provider.【F:core/controller/relay-controller.go†L53-L84】【F:core/middleware/distributor.go†L561-L623】

Caching occurs at multiple layers: Redis/memory response caching, per-channel
rate counters, and in-process request body reuse. Plugins share these caches via
the `meta.Meta` bag, avoiding redundant lookups as a request flows through the
chain. Together with adaptive timeouts and load-aware channel selection, these
mechanisms keep latency predictable while maximizing upstream utilization.
