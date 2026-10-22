# ADR-150: A Route Masks Its Sensitive Path Parameters in url.path Telemetry

**Status:** Accepted
**Date:** 2026-10-22

## Context

Two framework sinks write the concrete request path, not the route template:

- `overrideSpanURLPath` (`server/middleware.go`) replaces the incoming-request span's `url.path`
  attribute with `routerPath(r)`, the path the router keyed on (#1817).
- `extractRequestMetadata` (`server/logger.go`) sets the access log's `URLPath` to the same
  value, which feeds both the structured `url.path` field and the human message built by
  `createActionMessage`.

A path parameter carrying regulated data — a card number in `GET /cards/:pan` — therefore reached
the tracing exporter and the access log verbatim on every request to that route.

The log filter (`logger.SensitiveDataFilter`) cannot fix this. It masks by field NAME and never
inspects a value, and `pan` is deliberately absent from its defaults. Naming `url.path` in
`log.sensitivefields` would mask the path of every request on every route, destroying the field
for the routes that carry nothing sensitive, and would still leave the span attribute untouched,
since the filter only sees log records.

## Decision

**A route declares which of its path parameters are sensitive, and both path sinks mask exactly
those values for requests that route serves.**

- `server.WithSensitiveParams(names ...string)` is a `RouteOption`, following `WithTags`: it
  appends to the new `RouteDescriptor.SensitiveParams`, which `cloneDescriptor` deep-copies like
  `Tags` and `Middleware`.
- The lookup table is **per-server state**, not part of `DefaultRouteRegistry`. It lives on the
  `Server`'s `routeConflictTracker`, keyed by `{method, path}` (the full route template, as
  `c.Path()` reports it). `routeGroup.addEcho` records a typed route's `SensitiveParams` only once
  the conflict check admits it to the engine, so a refused duplicate declares nothing. The table is
  written during startup registration and read per request: it is copy-on-write behind an
  `atomic.Pointer`, so the read is lock-free, and a server that registered no sensitive route
  returns before any map lookup.
- `newServer` hands the tracker's lookup to the middleware chain before any route exists (the
  table fills afterwards). `maskSensitivePathParams(c, rawPath, lookup)` (`server/path_mask.go`)
  looks the matched route (`c.Request().Method`, `c.Path()`) up through it. With no sensitive
  parameters it returns `rawPath` itself and allocates nothing. Otherwise it lines the raw path's
  segments up with the route template's and replaces each declared `:name` segment with
  `logger.DefaultMaskValue`. When the segments do not line up — a `*` wildcard — it falls back to
  replacing the first literal occurrence of `c.Param(name)` in the path. The exported
  `SetupMiddlewares` has no routes of its own and passes no lookup.
- `overrideSpanURLPath` and `extractRequestMetadata` pass `routerPath(r)` through it. Masking the
  access log's `URLPath` covers its field and its message at once.
- `http.route` (`c.Path()`) is never touched in either sink: the template carries no value.

The design as first drafted replaced the first literal occurrence of the value everywhere. Segment
position replaced it as the primary strategy because a literal search can match the wrong text: a
`:pan` value of `1` under `/v1/cards/:pan` would have masked the `1` in `v1` and left the value
itself in the path.

## Alternatives considered

- **Add `url.path` to `log.sensitivefields`.** Rejected: it masks every route's path and never
  reaches the span.
- **Report the route template as `url.path`.** Rejected: `url.path` is the concrete path by
  semantic convention, and #1817 pinned it to the routed path so it agrees with `http.route`.
- **Detect PAN-shaped values automatically.** Rejected: value scanning is a heuristic with false
  positives and negatives on every path; the route author knows which parameter is regulated.
- **Linear scan of `ByPath` per request.** Rejected: the lookup sits on every request's hot path.
- **Side index on `DefaultRouteRegistry`.** The first implementation. Rejected: that registry is
  process-global and introspection-only, shared by every `Server` in the process, and tests
  `Clear()` it — which silently switched masking off for a live server, a security control that
  failed open. Its key also ignored the listener. Per-server state lives exactly as long as the
  routes it describes.

## Consequences

- **Additive** (`feat(server):`). A new field and a new option; no exported type or signature
  changes, and a route that declares nothing behaves exactly as before. `RouteDescriptor.
  SensitiveParams` stays in the registry for discovery only; masking never reads it there.
- Opt-in only: a consumer must declare every sensitive parameter on every route that carries one.
  Nothing detects an undeclared one.
- Wildcard routes fall back to a literal search; a value whose spelling in the path differs from
  `c.Param` is not found there and stays unmasked. Echo routes on the raw path and `c.Param`
  carries that raw spelling, so the motivating case — a numeric value on a parameterized route —
  is always masked by segment.
- Typed routes only (follow-up): a raw `RouteRegistrar.Add` route takes no `RouteOption`, so it
  cannot declare a sensitive parameter today.
- Only the framework's own path sinks are covered. A value a handler logs or sets on a span
  itself remains the consumer's responsibility.

## References

- `server/descriptor.go` — `RouteDescriptor.SensitiveParams`, `WithSensitiveParams`,
  `cloneDescriptor`
- `server/route_conflicts.go` — `routeConflictTracker.recordSensitive`, `.sensitiveParams`
- `server/route_registrar.go` — `routeGroup.addEcho`
- `server/path_mask.go` — `sensitiveParamLookup`, `maskSensitivePathParams`, `maskBySegment`
- `server/middleware.go` — `overrideSpanURLPath`
- `server/logger.go` — `extractRequestMetadata`
- `server/path_mask_test.go`, `server/descriptor_test.go`
- [observability.md](observability.md) — *Sensitive path parameters*
