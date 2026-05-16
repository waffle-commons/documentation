# Waffle Components Reference (Beta 0)

Below is the complete index of components shipped in the `waffle-commons` ecosystem at the `v0.1.0-beta0` release. Every component is an autonomous Git repository depending only on `waffle-commons/contracts` (plus any explicit additions declared in its own `composer.json`).

| Component | Package | Description | Reference |
| :--- | :--- | :--- | :--- |
| **Contracts** | `waffle-commons/contracts` | Root interface package. Every other component depends only on this. | [contracts.md](contracts.md) |
| **Core / Kernel** | `waffle-commons/waffle` | Framework facade — `AbstractKernel`, `ControllerArgumentResolver`, attribute dispatcher. | [core.md](core.md) |
| **Runtime** | `waffle-commons/runtime` | FrankenPHP worker bootstrap, classic SAPI fallback, `RuntimeInterface`. | [runtime.md](runtime.md) |
| **HTTP** | `waffle-commons/http` | PSR-7/17 implementation, `GlobalsFactory`, `ResponseEmitter`, trusted-hosts hardening. | [http.md](http.md) |
| **Routing** | `waffle-commons/routing` | `#[Route]` attribute router with route cache. | [routing.md](routing.md) |
| **Pipeline** | `waffle-commons/pipeline` | PSR-15 middleware stack and request handler. | [pipeline.md](pipeline.md) |
| **Security** | `waffle-commons/security` | ABAC engine, `#[Rule]` / `#[Voter]` attributes, CSRF (stateless double-submit). | [security.md](security.md) |
| **Container** | `waffle-commons/container` | PSR-11 container with autowiring and `ResettableInterface` for worker-mode reset. | [container.md](container.md) |
| **Event Dispatcher** | `waffle-commons/event-dispatcher` | PSR-14 dispatcher and listener provider. | [contracts.md#events-psr-14](contracts.md#events-psr-14) |
| **Log** | `waffle-commons/log` | PSR-3 `StreamLogger` (JSON, stdout/stderr), `LogChannel` enum-style constants. | [contracts.md#logging-psr-3](contracts.md#logging-psr-3) |
| **Cache** | `waffle-commons/cache` | PSR-6 + PSR-16 adapters: `ArrayCache`, `FileCache`, `RedisCache`, with stampede protection. | [cache.md](cache.md) |
| **Console** | `waffle-commons/console` | Zero-magic CLI runtime: `cache:clear`, `route:list`, `security:audit`. | [console.md](console.md) |
| **Config** | `waffle-commons/config` | Native YAML (ext-yaml) configuration loader with strict typing. | [config.md](config.md) |
| **Error Handler** | `waffle-commons/error-handler` | RFC 7807 JSON error renderer and PSR-15 middleware. | _Reference covered inline in [contracts.md](contracts.md)._ |
| **Utils** | `waffle-commons/utils` | Pure-function helpers shared across components (no I/O). | _Reference covered inline in [contracts.md](contracts.md)._ |

## Beta 0 contracts surface

The `waffle-commons/contracts` package is frozen as of Beta 0. Every interface is named `*Interface`, every exception ends in `*Exception`, every enum lives in an `Enum\` namespace. These conventions are enforced by `mago guard` in every component's `mago.toml`.

See [contracts.md](contracts.md) for the authoritative type listing.
