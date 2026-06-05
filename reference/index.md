# Waffle Components Reference (Beta-3)

Below is the complete index of components shipped in the `waffle-commons` ecosystem as of the in-progress `v0.1.0-beta3` release. Every component is an autonomous Git repository depending only on `waffle-commons/contracts` (plus any explicit additions declared in its own `composer.json`).

| Component | Package | Description | Reference |
| :--- | :--- | :--- | :--- |
| **Contracts** | `waffle-commons/contracts` | Root interface package. Every other component depends only on this. | [contracts.md](contracts.md) |
| **Core / Kernel** | `waffle-commons/waffle` | Framework facade — `AbstractKernel`, `ControllerArgumentResolver`, attribute dispatcher. | [core.md](core.md) |
| **Runtime** | `waffle-commons/runtime` | FrankenPHP worker bootstrap, classic SAPI fallback, `RuntimeInterface`. | [runtime.md](runtime.md) |
| **HTTP** | `waffle-commons/http` | PSR-7/17 implementation, `GlobalsFactory`, `ResponseEmitter`, trusted-hosts hardening. | [http.md](http.md) |
| **Routing** | `waffle-commons/routing` | `#[Route]` attribute router with route cache. | [routing.md](routing.md) |
| **Pipeline** | `waffle-commons/pipeline` | PSR-15 middleware stack and request handler. | [pipeline.md](pipeline.md) |
| **Security** | `waffle-commons/security` | Fail-closed ABAC engine, `#[Rule]` / `#[Voter]` / `#[PublicAccess]` attributes, stateless HMAC CSRF with `WAFFLE_SID` binding, `AnonymousSessionMiddleware`. | [security.md](security.md) |
| **HTTP Client** | `waffle-commons/http-client` | PSR-18 cURL client with `CURLOPT_PROTOCOLS` SSRF allowlist (HTTP/HTTPS only). | [http-client.md](http-client.md) |
| **Container** | `waffle-commons/container` | PSR-11 container with autowiring and `ResettableInterface` for worker-mode reset. | [container.md](container.md) |
| **Event Dispatcher** | `waffle-commons/event-dispatcher` | PSR-14 dispatcher and listener provider; `#[AsEventListener]` discovery. | [event-dispatcher.md](event-dispatcher.md) |
| **Log** | `waffle-commons/log` | PSR-3 `StreamLogger` (JSON, stdout/stderr), `LogChannel` enum-style constants. | [log.md](log.md) |
| **Cache** | `waffle-commons/cache` | PSR-6 + PSR-16 adapters: `ArrayCache`, `FileCache`, `RedisCache`, with stampede protection. | [cache.md](cache.md) |
| **Data** | `waffle-commons/data` | Worker-safe persistence (RFC-022): `PDOConnectionPool`, backend-agnostic query AST, SQL/Firestore compilers, property-hook hydrator, `db:migrate` migration runner. | [data.md](data.md) |
| **Console** | `waffle-commons/console` | Zero-magic CLI runtime: `cache:clear`, `route:list`, `security:audit`, `db:migrate`, `igor:audit`. | [console.md](console.md) |
| **Config** | `waffle-commons/config` | Native YAML (ext-yaml) configuration loader with strict typing. | [config.md](config.md) |
| **Error Handler** | `waffle-commons/error-handler` | RFC 7807 JSON error renderer and PSR-15 middleware. | [error-handler.md](error-handler.md) |
| **Utils** | `waffle-commons/utils` | Pure-function helpers shared across components (no I/O): `ClassParser`, `AttributeReader`, `ReflectionInspector`. | [utils.md](utils.md) |

## Developer tooling

| Tool | Description | Reference |
| :--- | :--- | :--- |
| **`wfl`** | Host-side developer CLI (`bin/wfl`) wrapping Docker / Composer / Mago / PHPUnit: lifecycle, per-component `mago` / `tests`, local component linking (`wfl link <consumer> <provider>`), and PHP debug/bench profile switching. | [wfl.md](wfl.md) |
| **Igor-PHP** | Worker-mode memory-neutrality gate — a static `ΔM = 0` audit (state mutation, incomplete `reset()`, dangerous globals) wired into the resident-state components (`runtime`, `container`, `data`, `security`, …). Run per component as `composer igor`, monorepo-wide as `./igor.sh` / `wfl igor`, or from the app console as `igor:audit` (engine in `runtime`). | [runtime.md](runtime.md) |

## Beta-3 data & persistence additions

Beta-3 introduces the **`waffle-commons/data`** component (RFC-022) — a worker-safe, ORM-free persistence layer — and the contracts that support it: `Waffle\Commons\Contracts\Data\Connection\ConnectionPoolInterface`, `Waffle\Commons\Contracts\Data\Exception\DatabaseExceptionInterface`, and `Waffle\Commons\Contracts\Data\Migration\MigrationRunnerInterface`. The `console` component gains the **`db:migrate`** command — it lives in `console` and depends only on the contracts interface, so no new dependency edge is introduced; the application wires the concrete `MigrationRunner` in its `bin/waffle`. See [data.md](data.md), [How to: Database Migrations](../how-to/database-migrations.md), and the [data CHANGELOG](../../data/CHANGELOG.md).

## Beta-2 contracts surface additions

Beta-2 adds the typed `405 Method Not Allowed` contract: `MethodNotAllowedException` (concrete `final` class) and its `MethodNotAllowedExceptionInterface` marker, the `Route` attribute's new `methods` parameter (relocated to `contracts`), and the `METHOD_GET` / `METHOD_POST` / … HTTP-method string constants on `Waffle\Commons\Contracts\Routing\Constant`. See the [contracts CHANGELOG](../../contracts/CHANGELOG.md) for the full Beta-2 delta and the umbrella [CHANGELOG](../../CHANGELOG.md) for the cross-component HTTP-correctness narrative.

## Beta-1 contracts foundations

Beta-1 made a **single intentional breaking change** to the contracts surface: `CsrfTokenManagerInterface::issue/validate/refresh` take a `$sessionId` argument so HMAC tokens bind to the per-browser `WAFFLE_SID`. It also added the `#[PublicAccess]` attribute, the concrete `RouteNotFoundException`, and CSRF binding constants (`SESSION_COOKIE_NAME`, `SESSION_ID_BYTES`, `SESSION_REQUEST_ATTRIBUTE`, `SESSION_COOKIE_MAX_AGE`).

Every interface is still named `*Interface`, every exception ends in `*Exception`, every enum lives in an `Enum\` namespace. These conventions are enforced by `mago guard` in every component's `mago.toml`. See [contracts.md](contracts.md) for the authoritative type listing and [attributes-public-access.md](attributes-public-access.md) for the new attribute.
