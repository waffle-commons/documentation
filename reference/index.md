# Waffle Components Reference (Beta-6)

Below is the complete index of components shipped in the `waffle-commons` ecosystem as of the `0.1.0-beta6` release. Every component is an autonomous Git repository depending only on `waffle-commons/contracts` — plus `waffle-commons/utils` where declared, and any explicit additions in its own `composer.json` — a perimeter enforced by `mago guard` (see [Architecture](../explanation/architecture.md)).

| Component | Package | Description | Reference |
| :--- | :--- | :--- | :--- |
| **Contracts** | `waffle-commons/contracts` | Root interface package. Every other component depends only on this. | [contracts.md](contracts.md) |
| **Core / Kernel** | `waffle-commons/waffle` | Framework facade — `AbstractKernel`, `ControllerArgumentResolver`, attribute dispatcher. | [core.md](core.md) |
| **Runtime** | `waffle-commons/runtime` | FrankenPHP worker bootstrap, classic SAPI fallback, `RuntimeInterface`. | [runtime.md](runtime.md) |
| **HTTP** | `waffle-commons/http` | PSR-7/17 implementation, `GlobalsFactory`, `ResponseEmitter`, trusted-hosts hardening. | [http.md](http.md) |
| **Routing** | `waffle-commons/routing` | `#[Route]` attribute router with route cache. | [routing.md](routing.md) |
| **Pipeline** | `waffle-commons/pipeline` | PSR-15 middleware stack and request handler. | [pipeline.md](pipeline.md) |
| **Security** | `waffle-commons/security` | Fail-closed ABAC engine, `#[Voter]` / `#[PublicAccess]` attributes, stateless HMAC CSRF with `WAFFLE_SID` binding, `AnonymousSessionMiddleware`. | [security.md](security.md) |
| **Auth** | `waffle-commons/auth` | Universal Authentication Bridge (RFC-021): OAuth2/OIDC + JWT (HS256/RS256, JWKS) + `X-Wfl-Assert-User` gateway assertions + API key/Basic, inbound middleware + outbound PSR-18 `AuthenticatedClient`; fail-closed, resettable `SecurityContext`. | [auth.md](auth.md) |
| Auth — WebAuthn / passkeys | `waffle-commons/auth` | WebAuthn passkey scheme (AUTH-01, RFC-021): `WebAuthnLibAdapter` is the sole `web-auth/webauthn-lib` importer; stateless authenticator with app-provided stores, fail-closed. | [webauthn.md](webauthn.md) |
| **HTTP Client** | `waffle-commons/http-client` | PSR-18 cURL client with `CURLOPT_PROTOCOLS` SSRF allowlist (HTTP/HTTPS only). | [http-client.md](http-client.md) |
| async | `waffle-commons/async` | Fiber-based finish-request task deferral (ASYNC-01, RFC-015): short post-response work drained on `TerminateEvent` under a bounded per-request budget — not a queue. | [async.md](async.md) |
| **Container** | `waffle-commons/container` | PSR-11 container with autowiring and `ResettableInterface` for worker-mode reset. | [container.md](container.md) |
| AOT Compilation | `waffle-commons/console` · `waffle-commons/routing` · `waffle-commons/waffle` | Ahead-of-Time build step (AOT-01/AOT-02, RFC-019): `container:compile` emits a graph-identical reflection-free container, `route:compile` a serialised O(depth) `RouteTrie`; opt-in via `WAFFLE_AOT=1` with reflection fallback. | [aot.md](aot.md) |
| **Event Dispatcher** | `waffle-commons/event-dispatcher` | PSR-14 dispatcher and listener provider; `#[AsEventListener]` discovery. | [event-dispatcher.md](event-dispatcher.md) |
| **Broadcast (Reactive)** | `waffle-commons/waffle` + contracts | Real-time state broadcasting (REACTIVE-01, RFC-018): `#[Broadcast]` property write-hooks record mutations into a request-scoped buffer, flushed over SSE on `TerminateEvent` — no I/O in the hook, worker-safe. | [broadcast.md](broadcast.md) |
| **Log** | `waffle-commons/log` | PSR-3 `StreamLogger` (JSON, stdout/stderr), `LogChannel` enum-style constants. | [log.md](log.md) |
| **Cache** | `waffle-commons/cache` | PSR-6 + PSR-16 adapters: `ArrayCache`, `FileCache`, `RedisCache`, with stampede protection. | [cache.md](cache.md) |
| **Telemetry** | `waffle-commons/telemetry` | SDK-free Prometheus observability (OBS-02, RFC-005): APCu-backed `MetricsRegistry`, stateless collectors, and the fail-closed `/waffle-metrics` middleware (bearer `hash_equals` + IP allow-list). | [telemetry.md](telemetry.md) |
| **Telemetry OTel** | `waffle-commons/telemetry-otel` | The sole OpenTelemetry-SDK importer (OBS-01, RFC-005): `TracerInterface` binding + W3C `traceparent` propagation; opt-in, keeps the SDK out of the core perimeter. | [telemetry-otel.md](telemetry-otel.md) |
| **Data** | `waffle-commons/data` | Worker-safe persistence (RFC-022): connection pools, the backend-agnostic query AST (SQR) with per-backend compilers, seven typed stateless repositories with the full CRUD write surface, live drivers, migrations, and OPcache warm-up. | [data.md](data.md) |
| Connection Pool | `waffle-commons/data` | Memory-resident pooling (DBAL-01/DBAL-02): heal-on-lease PDO/Redis pools (bounded, reset-rolls-back) plus the failsafe `TransactionIsolationMiddleware` for write requests. | [connection-pool.md](connection-pool.md) |
| **Console** | `waffle-commons/console` | Zero-magic CLI runtime: `cache:clear`, `route:list`, `security:audit`, `security:compare-audit`, `db:migrate`, `igor:audit`, `data:warmup`, the AOT `container:compile` / `route:compile` pair, and the nine `make:*` scaffolding makers (RFC-020). | [console.md](console.md) |
| **Config** | `waffle-commons/config` | Native YAML (ext-yaml) configuration loader with strict typing. | [config.md](config.md) |
| **Error Handler** | `waffle-commons/error-handler` | RFC 7807 JSON error renderer and PSR-15 middleware. | [error-handler.md](error-handler.md) |
| **Utils** | `waffle-commons/utils` | Pure-function helpers shared across components (no I/O): `Assert` (validation & cleansing), `ClassParser`, `AttributeReader`, `ReflectionInspector`. | [utils.md](utils.md) |

## Developer tooling

| Tool | Description | Reference |
| :--- | :--- | :--- |
| **`wfl`** | Host-side developer CLI (`bin/wfl`) wrapping Docker / Composer / Mago / PHPUnit: lifecycle, per-component `mago` / `tests`, local component linking (`wfl link <consumer> <provider>`), and PHP debug/bench profile switching. | [wfl.md](wfl.md) |
| **Igor-PHP** | Worker-mode memory-neutrality gate — a static `ΔM = 0` audit (state mutation, incomplete `reset()`, dangerous globals) wired into the resident-state components (`runtime`, `container`, `data`, `security`, …). Run per component as `composer igor`, monorepo-wide as `./igor.sh` / `wfl igor`, or from the app console as `igor:audit` (engine in `runtime`). | [runtime.md](runtime.md) |

## Beta-3 data & persistence additions

Beta-3 introduces the **`waffle-commons/data`** component (RFC-022) — a worker-safe, ORM-free persistence layer — and the contracts that support it: `Waffle\Commons\Contracts\Data\Connection\ConnectionPoolInterface`, `Waffle\Commons\Contracts\Data\Exception\DatabaseExceptionInterface`, and `Waffle\Commons\Contracts\Data\Migration\MigrationRunnerInterface`.

The Beta-3 cycle then completes the RFC-022 surface. In **contracts**: the SQR vocabulary (`Contracts\Data\Enum\{Operator, Direction}` — ⚠️ relocated from `Waffle\Commons\Data\Query` — plus the `QueryInterface` / `ComparisonInterface` / `OrderInterface` property-interfaces), the stateless `Contracts\Data\Repository\RepositoryInterface` (`find` / `findOne` / `stream`), and the CRUD write surface (`WritableRepositoryInterface`: `save` / `delete` / `findById`, through pure `DataMapperInterface` mappers). In **data**: all six relational dialects plus the MongoDB / key-value / Cassandra (CQL) / GraphQL compilers, seven typed repositories, live drivers, and the atomic flat-file JSON store — full type listing in [data.md](data.md), design rationale (ports-and-adapters drivers, the CQL transport situation, the workspace sandbox) in [Explanation: The Universal Data & Persistence Layer](../explanation/data-persistence.md). In **console**: `db:migrate` (concrete `MigrationRunner` wired by the app's `bin/waffle` — contracts-only edge, see [How to: Database Migrations](../how-to/database-migrations.md)), `data:warmup`, and the `make:entity` / `make:repository` makers (RFC-020) — see [console.md](console.md) and the [data CHANGELOG](../../data/CHANGELOG.md).

## Beta-6 additions

Beta-6 is the **stabilisation, audit and measurement** wave — it adds no new components. The contracts
surface changes in exactly two ways, both from the audit remediation:

- **Object-level ABAC (SEC-05).** New `Contracts\Security\SubjectResolverInterface` resolves the domain
  subject a route parameter identifies, so voters can express object-level (anti-IDOR) rules instead of
  reasoning about the request alone. Resolution is **lazy, voter-gated and fail-closed** — a
  `#[PublicAccess]` route with no voters never invokes it, and a resolver failure denies with 403. See
  [contracts.md](contracts.md), the [security reference](security.md), and
  [How to: Secure a Controller](../how-to/secure-a-controller.md).
- **⚠️ `#[PublicAccess]` is restricted to `Attribute::TARGET_METHOD`.** Class-level placement previously
  exempted *every* method of a controller — including methods added later — which is the failure mode the
  attribute exists to prevent. Class-level usage is now rejected by PHP when the attribute is
  read; annotate each public method explicitly. See [attributes-public-access.md](attributes-public-access.md) and
  [Explanation: Fail-Closed ABAC](../explanation/security-fail-closed-abac.md).

The remaining fifteen gate-blocking findings were fixed **inside** the components without changing their
public contracts — SQL/CQL identifier quoting on write paths, Maker codegen injection, route-cache
deserialization, a `BasicAuthenticator` timing oracle, the HS* secret length floor, upload-path
containment, fail-secure YAML parsing, escape-by-default controller string returns, validate-before-cast
route parameters, and AOT/interpreted container reset parity. Per-component detail lives in each
CHANGELOG; the measured runtime numbers are in
[Explanation: Performance Strategy](../explanation/performance.md).

## Beta-5 additions

Beta-5 is the **performance, observability & real-time** wave (RC-readiness groundwork) — see [Explanation: Performance Strategy](../explanation/performance.md) for how these pieces compound. Six feature areas land:

- **Connection pooling (Beta-5, DBAL-01/DBAL-02).** Memory-resident DB pooling for FrankenPHP resident workers — heal-on-lease PDO/Redis pools that roll back stragglers on reset, plus a failsafe transaction middleware for write requests: [why & how it works](../explanation/connection-pooling.md), the [Connection Pool reference](connection-pool.md), and [how to configure it](../how-to/configure-connection-pooling.md).
- Beta-5 ships contract-first observability (RFC-005): the new SDK-free [`waffle-commons/telemetry`](telemetry.md) (Prometheus `/waffle-metrics`, stateless collectors, APCu-backed `MetricsRegistry`) and the opt-in [`waffle-commons/telemetry-otel`](telemetry-otel.md) OpenTelemetry bridge — the tracing/metrics contracts and their no-op defaults live in [contracts](contracts.md), so the framework core never imports an SDK; see [Explanation: Contract-First Observability](../explanation/observability-telemetry.md) and [How to: Enable Telemetry and Metrics](../how-to/enable-telemetry-and-metrics.md).
- **Async (ASYNC-01 / ASYNC-02, RFC-015).** New `waffle-commons/async` ships the Fiber-based finish-request deferral runner (`DeferredTaskRunner` behind `Contracts\Async\TaskRunnerInterface`) that lifts short post-response work off the latency path under a bounded per-request budget — see the [async reference](async.md), the [explanation](../explanation/async-finish-request-deferral.md), and the [how-to: defer post-response work](../how-to/defer-post-response-work.md); the concurrency half (concurrent `sendRequests()` / `promise()` fan-out) lands in `waffle-commons/http-client`'s `ConcurrentClientInterface`.
- **WebAuthn / passkeys (AUTH-01, AXE6)** — the passkey scheme of the Universal Authentication Bridge lands in `waffle-commons/auth`: a single audited adapter behind `Contracts\Auth\WebAuthn\*`, stateless across worker requests with two app-provided stores. See [Explanation: WebAuthn & Passkeys](../explanation/webauthn-passkeys.md), [Reference: WebAuthn / passkeys](webauthn.md), and [How to: Register and Verify Passkeys](../how-to/register-and-verify-passkeys.md).
- **Ahead-of-Time compilation (AOT-01 / AOT-02, RFC-019).** Build-time container + route compilation for sub-millisecond cold starts under FrankenPHP — see [Explanation: Ahead-of-Time Compilation](../explanation/aot-compilation.md), the [AOT reference](aot.md), and [How to: Enable AOT](../how-to/enable-aot.md).
- Beta-5 adds **reactive state broadcasting** (REACTIVE-01, RFC-018, AXE3): `#[Broadcast(channel)]` write-hooks record mutations into a request-scoped buffer with no I/O, flushed over SSE after the response cycle. See [Explanation: Reactive State Broadcasting](../explanation/reactive-broadcast.md), the [broadcast reference](broadcast.md), and [How to: Broadcast State Changes](../how-to/broadcast-state-changes.md).

## Beta-4 additions

Beta-4 is the **security-hardening & worker-mode-diagnostics** wave (RC-readiness groundwork). New contracts: `Contracts\Validation\SelfValidatingInterface` (behind the mockable `ValidatorInterface`), `Contracts\Data\Connection\ConnectionTrackerInterface` + `Contracts\Data\Connection\ConnectionKind` (the orphaned-connection tracer port), and `Contracts\Handler\ResponseFactoryAwareInterface` (interface-based response-factory injection). Diagnostics: `container`'s boot-time `ComplianceScanner` (DIAG-02) and `runtime`'s `ConnectionTracker` feeding `waffle`'s `OrphanedConnectionListener` (DIAG-03). Security: SSRF is **default-on** in `http-client` (SEC-02), fail-closed CORS in `security` (SEC-04), and the `security:compare-audit` / `wfl compare-audit` timing-safety gate in `console` (SEC-03). DX: `wfl check:all` / `wfl monorepo:sync`, native `mb_trim` (DX-04), and the `utils` `AssertValidator` (DX-05). See the per-component CHANGELOGs and the [root CHANGELOG](../../CHANGELOG.md).

## Beta-2 contracts surface additions

Beta-2 adds the typed `405 Method Not Allowed` contract: `MethodNotAllowedException` (concrete `final` class) and its `MethodNotAllowedExceptionInterface` marker, the `Route` attribute's new `methods` parameter (relocated to `contracts`), and the `METHOD_GET` / `METHOD_POST` / … HTTP-method string constants on `Waffle\Commons\Contracts\Routing\Constant`. See the [contracts CHANGELOG](../../contracts/CHANGELOG.md) for the full Beta-2 delta and the umbrella [CHANGELOG](../../CHANGELOG.md) for the cross-component HTTP-correctness narrative.

## Beta-1 contracts foundations

Beta-1 made a **single intentional breaking change** to the contracts surface: `CsrfTokenManagerInterface::issue/validate/refresh` take a `$sessionId` argument so HMAC tokens bind to the per-browser `WAFFLE_SID`. It also added the `#[PublicAccess]` attribute, the concrete `RouteNotFoundException`, and CSRF binding constants (`SESSION_COOKIE_NAME`, `SESSION_ID_BYTES`, `SESSION_REQUEST_ATTRIBUTE`, `SESSION_COOKIE_MAX_AGE`).

Every interface is still named `*Interface`, every exception ends in `*Exception`, every enum lives in an `Enum\` namespace. These conventions are enforced by `mago guard` in every component's `mago.toml`. See [contracts.md](contracts.md) for the authoritative type listing and [attributes-public-access.md](attributes-public-access.md) for the new attribute.
