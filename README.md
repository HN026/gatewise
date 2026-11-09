<p><img src="assets/logo.svg" alt="Gatewise logo" width="80" height="80" /></p>

# Gatewise

When a dependency slows down, requests take longer and more work stays in flight. Gatewise bounds concurrent work, reserves capacity for important requests, and can adjust admission using latency, errors and configured resource signals.

Use it as Go middleware, as a standalone HTTP proxy for services written in any language, or around Go worker jobs and SQL operations. Limits apply to one engine in one process.

[Documentation](https://hn026.github.io/gatewise-site/) · [Installation](https://hn026.github.io/gatewise-site/#/installation) · [Examples](https://hn026.github.io/gatewise-site/#/examples) · [Benchmarks](https://hn026.github.io/gatewise-site/#/benchmarks) · [Versions](https://hn026.github.io/gatewise-site/#/versions)

## Choose an integration

| Integration | What it protects | Where to use it |
| --- | --- | --- |
| Go middleware | Synchronous HTTP handlers, until they return | Inside a Go service |
| HTTP proxy | Active proxy requests | In front of an HTTP service in any language |
| Operation engine | Work until its callback returns or its permit is finished | Go workers and other explicitly owned work |
| SQL adapter | Statements, row streams and transactions through their documented lifetimes | Around an existing `database/sql` pool |

A fixed concurrency ceiling is the starting point. Add priorities, tenant policies, adaptation or waiting only when the workload needs them. A classless semaphore may be sufficient for a simple ceiling; a rate limiter serves a different purpose by enforcing requests per time interval. See the [policy selection and tuning guide](https://hn026.github.io/gatewise-site/#/installation/choose-policy).

## Go middleware

Requires Go 1.22 or newer. Use a currently supported, patched Go toolchain for deployment.

```sh
go get github.com/HN026/gatewise@v1.0.0
```

Add a guard to routes that share capacity:

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

This is an integration fragment: `mux` is your `http.ServeMux`, `checkout` and `reindex` are your handlers, and the enclosing application imports `log` and `net/http`. After adding the integration, run `go mod tidy`, then build or start your service. A [complete application example](https://hn026.github.io/gatewise-site/#/installation) is available in the installation guide.

Here, at most 20 protected requests execute at once across both routes. Background work can use at most 15 slots, leaving five reserved for critical work. These are configurable example values, not a requests-per-second quota. Reuse one guard for routes sharing that budget. Excess requests receive structured HTTP 503 responses. Optional waiting can hold eligible requests briefly before execution capacity becomes available.

## Standalone HTTP proxy

Build the released proxy:

```sh
git clone --branch v1.0.0 https://github.com/HN026/gatewise.git
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

Addresses and limits are examples. Point `upstream_url` at your running backend. Validate the configuration, start the proxy, then check a route:

```sh
./bin/gatewise -check-config -config configs/orders.json -env-file ""
./bin/gatewise -config configs/orders.json -env-file ""
```

In another terminal:

```sh
curl -i http://127.0.0.1:8080/healthz
curl -i http://127.0.0.1:8080/orders/hello
```

The second request forwards `/hello` to your backend; its response depends on that handler. Routing uses the longest matching path-segment prefix. Add entries under `services` and `routes` to protect multiple backends.

Configuration is resolved at startup, with precedence: process environment > dotenv > JSON > defaults. The commands above disable automatic dotenv loading with `-env-file ""`; process environment overrides still apply. Configuration validation opens no listener and does not contact upstreams. See the [configuration reference](https://hn026.github.io/gatewise-site/#/reference) for environment variables and policy settings.

### Run with Docker

The published proxy image supports `linux/amd64` and `linux/arm64`:

```sh
docker pull ghcr.io/hn026/gatewise:v1.0.0
docker run --rm --name gatewise --read-only --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --add-host=host.docker.internal:host-gateway \
  -p 127.0.0.1:8080:8080 \
  -e GATEWISE_SERVICE_ORDERS_URL=http://host.docker.internal:9090 \
  ghcr.io/hn026/gatewise:v1.0.0
```

The address is an example: use a backend reachable from the container. A container’s `127.0.0.1` is its own loopback; on Linux, a host backend bound only to loopback is not reachable through the bridge gateway. For services on the same container network, use the backend’s service name.

The supplied configuration listens on `0.0.0.0:8080` and forwards `/orders`. Mount your own JSON at `/etc/gatewise/config.json:ro` for other routes or multiple services. Pass runtime variables with Docker’s `-e` or `--env-file`; automatic application `.env` loading is disabled. The image runs as UID/GID 65532 and contains a static binary and CA roots, with no shell or Go toolchain.

To build locally from the cloned repository:

```sh
docker build --build-arg VERSION=v1.0.0 -t gatewise:v1.0.0 .
```

`docker stop --time 10 gatewise` allows the default five-second graceful shutdown. The [Docker guide](https://hn026.github.io/gatewise-site/#/installation/docker) covers mounted configuration, Podman, image digests, SBOMs and provenance. Middleware users include Gatewise in their own application image.

## Worker jobs and SQL operations

`operation.New` creates an admission engine without HTTP. Use `Engine.Do(ctx, request, fn)` around a job, or `Begin`, `Permit.Context` and `Finish` when the caller owns the lifetime explicitly. Cancellation does not release work that is still running. Use separate engines for distinct HTTP, worker and database budgets; nesting the same engine through its active context returns `operation.ErrNested`.

`sqlguard.New(db, engine, metadata)` wraps an existing `database/sql` pool:

- Statements hold admission through completion.
- Row streams hold admission through `Close` or terminal iteration.
- Transactions hold admission through `Commit` or `Rollback`, including driver cleanup.

Always close rows and use transaction methods with `tx.Context()` inside `InTx`. The application owns the driver, pool and driver/network timeouts. See the [operation and SQL installation guide](https://hn026.github.io/gatewise-site/#/installation/operation-setup) and [runnable SQL/worker example](https://hn026.github.io/gatewise-site/#/examples/operation-example).

## Optional admission policies

Existing integrations retain their defaults. Immediate refusal, fixed concurrency and two-class critical/background admission are available without configuring the policies below.

| Policy | What it adds | Setup |
| --- | --- | --- |
| Adaptive concurrency | Adjusts the execution limit within configured bounds using completed-work feedback | [Adaptive profiles](https://hn026.github.io/gatewise-site/#/installation/stable-adaptive) |
| Priority tiers and propagation | Ordered tiers, route defaults and trusted downstream metadata | [Priority integration](https://hn026.github.io/gatewise-site/#/installation/priority-propagation) |
| Tenant isolation | Per-tenant concurrency caps with bounded identity tracking | [Tenant integration](https://hn026.github.io/gatewise-site/#/installation/tenant-isolation) |
| Tenant sharing | Target shares and bounded borrowing of spare capacity | [Allocation rules](https://hn026.github.io/gatewise-site/#/installation/tenant-sharing) |
| Resource signals | Cached overload observations with explicit freshness and failure policies | [Signal setup](https://hn026.github.io/gatewise-site/#/installation/signal-setup) |
| Resource budgets | Planned logical write-byte costs and concurrent work per bounded, hashed key | [Resource setup](https://hn026.github.io/gatewise-site/#/installation/resource-setup) |
| Bounded waiting | A separate, deadline-aware waiting budget when execution capacity is temporarily unavailable | [Queue setup](https://hn026.github.io/gatewise-site/#/installation/bounded-waiting) |

### Adaptive concurrency

Adaptive mode is opt-in. Existing adaptive integrations retain the `legacy` profile; choose `Adaptive: &middleware.AdaptiveConfig{Profile: "stable"}` for the stable profile, or the corresponding `operation.AdaptiveConfig`. The proxy uses an enabled `adaptive` configuration with `profile: "stable"`. Explicit bounds and settings remain authoritative.

The stable profile requires healthy latency headroom before growth, uses local capacity refusals as demand evidence, and protects sufficiently sampled priority-tier tails. Matching resource ceilings can veto growth. Snapshots, adjustment logs and metrics explain adjustments and holds. Configured floors, intrinsic operation latency and shared dependencies can still prevent a latency target from being met. See the [profile rules and examples](https://hn026.github.io/gatewise-site/#/installation/stable-adaptive).

### Trusted identity, priorities and signals

Resolve tenants and priorities from authenticated, server-owned information. The proxy trusts configured identity/priority metadata only from configured immediate peers; ingress must authenticate callers and overwrite reserved headers. Forwarded IP headers do not establish trust. Go integrations use `priority.With` and an explicitly allowlisted `priority.Transport` for propagation.

Tenant tracking is bounded and idle entries expire; active entries are never evicted. Hard tenant caps remain the default. Optional sharing permits borrowing up to separate total ceilings while protecting active/recent demand. Priority cohorts provide consistent ordering within an epoch, not identical admission decisions across services or a starvation guarantee.

`regulator` providers can observe Go runtime pressure, SQL-pool utilization and new SQL-pool contention. `postgres.NewReplication` observes one named physical standby from a separate primary monitoring pool. These observations are application adapters, not database-engine enforcement or cluster-wide limits. Start signal decisions in observe mode and choose freshness and unavailable-data behavior before enforcement. Resource costs must come from a trusted operation planner.

### Bounded waiting

Waiting is disabled by default. For example:

```go
Queue: &middleware.QueueConfig{
    MaxWaiting: 32, MaxWait: 40 * time.Millisecond,
    AgingAfter: 20 * time.Millisecond,
    Reserved: map[middleware.Priority]int{middleware.Critical: 8},
},
```

These are example values. Waiting and execution are separate budgets. Only local capacity refusals wait; every selected request rechecks policy and holds no execution, tenant or resource permit while waiting. FIFO within a tier, priority preference and aging determine eligibility; hard execution reservations still apply. An earlier caller deadline wins. Full, reserved and expired waits have explicit rejection reasons.

HTTP waiting buffers eligible known-length bodies, by default up to 64 KiB, so HTTP/1.1 disconnects can be observed. Configure `http.Server.ReadTimeout` to bound upload preparation; proxy buffering requires a positive `read_timeout`. Large or unknown-length bodies stream and fail fast instead of waiting. See the [body buffering and opt-out rules](https://hn026.github.io/gatewise-site/#/installation/bounded-waiting) before enabling queues.

Waiting does not add execution capacity or guarantee lower-tier progress. Queues are in memory, process-local and non-durable. Queue time is reported separately from processing feedback; assess useful completions against the whole response budget.

## Metrics and operations

The proxy exposes `/healthz` and Prometheus `/metrics` outside admission. Health reports proxy liveness, not backend availability. Go applications register guards or operation engines with `metrics.Register` and their own Prometheus registry. A Grafana dashboard and optional local monitoring stack are in `deploy/observability/`.

`ControllerSnapshot()` exposes evaluated samples, latency ratio, failure fraction and the last decision, including holds. Use metrics to distinguish execution pressure, waiting, tenant refusals and binding resource signals. HTTP and operation counts describe different layers; adding them does not give end-to-end throughput.

Applications own authentication, TLS, retry policy, timeouts and graceful shutdown. Stop intake before shutdown, give active work time to finish, and cancel collectors and close database resources in the application. Providers must honor cancellation; an uncooperative provider can retain one worker per signal. See [reading metrics](https://hn026.github.io/gatewise-site/#/metrics/reading-metrics) and [deployment, rollout and rollback](https://hn026.github.io/gatewise-site/#/installation/operating).

## Guarantees and boundaries

- Limits and queues belong to one engine in one process. Additional replicas multiply independent budgets unless you size them against shared dependency capacity.
- Middleware holds admission until its synchronous handler returns. A canceled proxy request can finish while remote work is still cleaning up. Use operation or SQL guards where that backend lifetime needs protection.
- There is no preemption. Cancellation cannot force arbitrary running code or a driver to stop; permits remain held through their protected lifetime.
- Reuse an engine for work sharing a budget. Do not copy constructed services, engines, permits or SQL adapters. Always finish permits and close rows.
- Adaptation operates within configured bounds; it does not discover a universal production ceiling. Long-lived streaming and WebSockets are outside the current scope.

## Demo example and benchmarks

The separate [documentation repository](https://github.com/HN026/gatewise-site) provides a source-only commerce example: orders, catalog and inventory services backed by PostgreSQL, with Postman collections. It supports middleware and proxy deployments. [Download and run the demo example](https://hn026.github.io/gatewise-site/#/examples).

The [benchmarks](https://hn026.github.io/gatewise-site/#/benchmarks) explain the problem, comparison, results and tradeoffs for each experiment. Reports retain successful work within a latency budget, successful-request latency, refusals, failures, pool waiting and recovery. The v1 sustained protocol passed eight three-minute runs across local and Linux environments: 1,232,800 requests and 120 checks. The separate short exploratory recovery miss remains visible.

These controlled experiments do not establish production capacity or a Cinnamon performance comparison. Use the [matched policy comparison](https://hn026.github.io/gatewise-site/#/benchmarks/hardening) to assess the value beyond a semaphore, and the [sustained validation](https://hn026.github.io/gatewise-site/#/benchmarks/reliability) to inspect cleanup and recovery. The older synthetic HTTP runner remains in `benchmarks/` for custom fixtures.

## Versions and compatibility

Install a tagged release and review its [upgrade notes](https://hn026.github.io/gatewise-site/#/versions). The v1 contract covers the exported `decision`, `metrics`, `middleware`, `operation`, `postgres`, `priority`, `regulator` and `sqlguard` APIs, documented proxy configuration, CLI flags and HTTP rejection fields. `internal/` packages are not public APIs.

Use constructors and keyed configuration literals. v1.x may add optional fields, diagnostic reasons and metrics; accept unknown response fields and avoid depending on exact error or log text. CI checks the frozen exported API and runs race tests on Linux AMD64/ARM64 with Go 1.22 and current stable Go. See the [compatibility contract](https://hn026.github.io/gatewise-site/#/reference/compatibility).

The [development timeline](https://hn026.github.io/gatewise-site/#/versions/development) groups the work across approximately 15 October–9 November 2026. Its milestone dates are approximate; the [release history](https://hn026.github.io/gatewise-site/#/versions/release-history) records actual publication dates. Published tags and validation records retain their original identities.

## Development

```sh
go test -race ./...
go vet ./...
go build ./cmd/gatewise
```

Public integrations live in the packages listed above. Executable wiring is in `cmd/`, implementation details are in `internal/`, and focused tests are under `tests/`. The database-backed demo and Postman bundles are maintained in the documentation repository.

## License

MIT. Gatewise is an independent project inspired by published overload-control techniques; it is not affiliated with Uber.
