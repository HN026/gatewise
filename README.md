<p><img src="assets/logo.svg" alt="Gatewise logo" width="80" height="80" /></p>

# Gatewise

HTTP admission control for Go applications and HTTP backends. Gatewise bounds concurrent requests, reserves capacity for critical traffic, and optionally adjusts admission using latency and error feedback.

Use it as Go middleware or as a standalone proxy in front of an HTTP service written in any language.

[Documentation](https://hn026.github.io/gatewise-site/) · [Installation](https://hn026.github.io/gatewise-site/#/installation) · [Examples](https://hn026.github.io/gatewise-site/#/examples) · [Benchmarks](https://hn026.github.io/gatewise-site/#/benchmarks)

## Go middleware

Requires Go 1.22 or newer.

```sh
go get github.com/HN026/gatewise@v0.5.0
```

```go
import "github.com/HN026/gatewise/middleware"

// Example values: choose limits by load testing your service.
guard, err := middleware.NewService(middleware.ServiceConfig{
    Name: "orders", MaxConcurrent: 20, ReservedCritical: 5,
})
if err != nil {
    log.Fatal(err)
}
mux.Handle("POST /checkout", guard.Middleware(
    middleware.Critical, http.HandlerFunc(checkout),
))
mux.Handle("POST /reindex", guard.Middleware(
    middleware.Background, http.HandlerFunc(reindex),
))
```

This is an integration fragment: `mux` is your `http.ServeMux` and `checkout`/`reindex` are application handlers. See the [complete example](https://hn026.github.io/gatewise-site/#/installation).

After adding the imports and handlers to your application, run `go mod tidy` to resolve the packages it uses, then build or start your service.

The example allows 20 requests running simultaneously across both routes, with background work limited to 15. This is concurrency, not requests per second. Reuse one guard for routes sharing capacity. Excess requests receive structured HTTP 503 responses. Limits are configurable and apply per process.

## Standalone proxy

```sh
git clone --branch v0.5.0 https://github.com/HN026/gatewise.git
cd gatewise
go build -o bin/gatewise ./cmd/gatewise
```

Save a configuration such as `configs/orders.json`:

```json
{
  "listen_addr": "127.0.0.1:8080",
  "services": {
    "orders": {
      "upstream_url": "http://127.0.0.1:9090",
      "max_concurrent": 20,
      "reserved_critical": 5
    }
  },
  "routes": [
    {"path_prefix": "/orders", "service": "orders", "strip_prefix": true, "priority": "critical"},
    {"path_prefix": "/orders/background", "service": "orders", "strip_prefix": true, "priority": "background"}
  ]
}
```

Addresses and limits above are examples. Point `upstream_url` at your running backend, then start:

```sh
./bin/gatewise -config configs/orders.json -env-file ""
curl -i http://127.0.0.1:8080/healthz
curl -i http://127.0.0.1:8080/orders/hello
```

The second request forwards `/hello` to your backend; its response depends on that handler. Routing uses the longest matching path-segment prefix. Configuration precedence is process environment > dotenv > JSON > defaults. Configuration is resolved at startup. See the [reference](https://hn026.github.io/gatewise-site/#/reference) for adaptive settings, route budgets and environment variables.

## Tenant isolation

Optional `ServiceConfig.Tenants` adds per-tenant concurrency caps inside each service budget. Supply a resolver using your authenticated principal. The proxy accepts a dedicated identity header only from configured immediate-peer CIDRs; the ingress must authenticate and overwrite it. Forwarded headers do not establish trust.

Tracking has a configurable bound and idle expiry. Missing identities and registry exhaustion reject by default; explicit bounded shared-bucket policies are available. Active entries are never evicted. Tenant rejections do not feed adaptive latency or failure observations. Limits apply per guard and process, not across replicas.

`guard.TenantSnapshot()` and aggregate Prometheus metrics expose tracking and rejection reasons without identity labels. See the [tenant integration guide](https://hn026.github.io/gatewise-site/#/installation/tenant-isolation) and [Postman example](https://hn026.github.io/gatewise-site/#/examples/tenant-isolation).

## Metrics and operations

The proxy exposes `/healthz` and Prometheus `/metrics` outside admission. Go applications can register a guard using `metrics.Register` with their own Prometheus registry. A Grafana dashboard and optional local monitoring stack are included under `deploy/observability/`.

Adaptive consumers can inspect `guard.ControllerSnapshot()` for the last evaluated sample count, latency ratio, failure fraction and decision, including holds. The Prometheus collector and Grafana dashboard expose the same observations.

Validate proxy configuration offline before deployment:

```sh
./bin/gatewise -check-config -config configs/orders.json -env-file ""
```

This exits without opening a listener or contacting upstreams. It does not verify backend availability.

Slots are held until synchronous handlers return. There is no admission queue or preemption. Adaptive mode uses completed-request feedback within configured bounds; it does not directly measure database capacity. Multiple replicas have independent budgets. Applications own retry policy, dependency timeouts, authentication and TLS. Long-lived streaming and WebSocket support are outside the current scope.

## Demo example and measurements

The separate [documentation repository](https://github.com/HN026/gatewise-site) provides a source-only commerce example: orders, catalog and inventory services, three PostgreSQL databases, and a portable Postman collection. Both proxy and middleware modes are supported. [Download and run the demo example](https://hn026.github.io/gatewise-site/#/examples).

[Published benchmarks](https://hn026.github.io/gatewise-site/#/benchmarks) report sustained SQL traffic, overload, injected database failures and recovery. Results separate successful work within a latency budget, successful-request p95, rejection rates and SQL-pool waits. These controlled local experiments do not establish production capacity. The older synthetic runner remains available in `benchmarks/` for custom HTTP fixtures.

## Development

```sh
go test -race ./...
go vet ./...
go build ./cmd/gatewise
```

Public packages are `middleware/` and `metrics/`. Executable wiring is in `cmd/`; implementation packages are in `internal/`; focused tests are under `tests/`.

## License

MIT. Gatewise is an independent project inspired by published overload-control techniques; it is not affiliated with Uber.

### Priority propagation

Optional `ServiceConfig.Priorities` adds 2–8 ordered tiers, compatibility mappings for Critical/Background, trusted HTTP metadata and optional rotating cohorts. Use `priority.With` after authentication and `priority.Transport` with an explicit downstream host allowlist. Missing metadata uses the route default; untrusted supplied metadata returns 403. Cohorts provide consistent ordering within a UTC epoch, not identical admission decisions across services or starvation guarantees. Existing services without tier configuration retain two-class admission. No queue or capacity borrowing is added.

[Priority setup](https://hn026.github.io/gatewise-site/#/installation/priority-propagation) · [Runnable service chain](https://hn026.github.io/gatewise-site/#/examples/priority-chain)

### Tenant sharing

Optional `TenantConfig.Sharing` turns tenant caps into target shares and allows identified tenants to borrow up to separate total ceilings. Active/recent demand protects other tenants’ shares; targets scale down when the adaptive limit is smaller, and returning below-share work can pause new borrowing. Running requests finish normally. Hard caps remain the default. [Setup and allocation rules](https://hn026.github.io/gatewise-site/#/installation/tenant-sharing).
