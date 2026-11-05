# Cache Reference (`waffle-commons/cache`)

> **Release:** `0.1.0-beta6` &nbsp;|&nbsp; *Adds the optional dev-only connection-tracer hook (DIAG-03)*
> **PSR Compliance:** PSR-6 (`Psr\Cache\CacheItemPoolInterface`), PSR-16 (`Psr\SimpleCache\CacheInterface`)

PSR-6 + PSR-16 cache implementation tuned for FrankenPHP worker mode. Every adapter is stateless across requests, fail-secure, and zero-baseline under Mago.

## Adapters

| Class | Backend | Worker scope | Notes |
| :--- | :--- | :--- | :--- |
| `Waffle\Commons\Cache\Adapter\ArrayCache` | In-memory PHP array | Single worker request cycle | Ultra-fast; lifetime = worker request. |
| `Waffle\Commons\Cache\Adapter\FileCache` | Filesystem | Per host | Strict permission handling, no `/tmp` foot-guns. |
| `Waffle\Commons\Cache\Adapter\RedisCache` | Redis (via `predis/predis`) | Cluster-wide | Shared across multiple workers / hosts. |

All adapters implement `Waffle\Commons\Contracts\Cache\CacheInterface` (which extends PSR-16's `Psr\SimpleCache\CacheInterface`).

## Pool (PSR-6 view)

`Waffle\Commons\Cache\Pool\CachePool` wraps any `CacheInterface` adapter and exposes the PSR-6 `CacheItemPoolInterface` API on top of it. Items are represented by `Waffle\Commons\Cache\Pool\CacheItem`.

## Factory — `Waffle\Commons\Cache\Factory\CacheFactory`

Builds a concrete adapter from a backend identifier and a flat options array. Kept parameter-driven (no `ConfigInterface` dependency) so the `cache` component stays agnostic of how the host application stores configuration.

```php
public function create(string $adapter, array $options = []): CacheInterface
```

Backend identifiers — exact constants from `Waffle\Commons\Contracts\Cache\Constant`:

| Identifier | Adapter | Required options | Optional options |
| :--- | :--- | :--- | :--- |
| `Constant::BACKEND_ARRAY` | `ArrayCache` | — | `default_ttl: int` |
| `Constant::BACKEND_FILE` | `FileCache` | `directory: string` | `default_ttl: int` |
| `Constant::BACKEND_REDIS` | `RedisCache` | — | `dsn: string` (default `redis://localhost:6379`), `prefix: string`, `default_ttl: int` |

Unknown identifier → `\InvalidArgumentException`. Backend construction failure → `Waffle\Commons\Cache\Exception\CacheBackendUnavailableException`.

```php
public const float REDIS_TIMEOUT = 1.0;
```

On its own, Predis waits 5 s to connect and PHP's `default_socket_timeout` (60 s) to read, so the cache's `/readyz` check could outlast the readiness probe. `CacheFactory` sets both to `REDIS_TIMEOUT` seconds; a DSN that sets its own (`?timeout=…&read_write_timeout=…`) keeps them (*since `0.1.0-beta7`*, RFC-023 §3.2).

## Connection tracing (DIAG-03)

`CacheFactory` accepts an optional dev-only `?Waffle\Commons\Contracts\Data\Connection\ConnectionTrackerInterface`; when present it is threaded into `RedisCache`, which registers its client (`ConnectionKind::Redis`) so the runtime's orphaned-connection tracer can report a Redis socket still open at request end. In production the tracker is `null` and the hook is a zero-cost no-op. See [runtime.md → Orphaned-connection tracer](runtime.md#orphaned-connection-tracer--connectiontracker-diag-03).

## Key validation

`Waffle\Commons\Cache\KeyValidator` enforces the PSR-16 key character set. Invalid keys throw `Waffle\Commons\Cache\Exception\InvalidCacheKeyException`.

## Stampede protection

`Waffle\Commons\Cache\Trait\StampedeAwareTrait` implements probabilistic early-expiration ("XFetch") to prevent thundering-herd misses under high concurrent load. Adapters that wish to opt in declare `implements StampedeProtectionInterface` and `use StampedeAwareTrait;`.

## Exceptions

All inherit from `Waffle\Commons\Cache\Exception\CacheException` and implement `Waffle\Commons\Contracts\Cache\Exception\CacheExceptionInterface`:

- `CacheBackendUnavailableException` — implements `CacheBackendUnavailableExceptionInterface`. Raised when an adapter's backing store cannot be reached.
- `InvalidCacheKeyException` — implements `InvalidCacheKeyExceptionInterface` + PSR-16's `Psr\SimpleCache\InvalidArgumentException`. Raised by `KeyValidator`.

## Readiness probe — `Waffle\Commons\Cache\Health\CacheHealthCheck`

*Added in `0.1.0-beta7` (OPS-01).* A `HealthCheckInterface` for `/readyz`: one `has()` on a fixed key
(`PROBE_KEY = 'waffle.health.probe'`) — a single read round trip, never a write. A Redis-backed cache that
cannot reach Redis throws, and the probe reports it down.

```php
final readonly class CacheHealthCheck implements HealthCheckInterface
{
    public const string PROBE_KEY = 'waffle.health.probe';

    public function __construct(
        private CacheInterface $cache,
        private string $name = 'cache',
        private bool $critical = false,
    ) {}
}
```

Not critical by default: an application that cannot serve without its cache says so (pass
`critical: true`, or list `cache` under `waffle.health.required`). The skeleton registers it only when
`waffle.cache.adapter` is `redis` — an in-memory cache would always answer "up". Give it the raw cache, not
the `MeteredCache` decorator, or every probe skews the hit ratio. See
[pipeline.md](pipeline.md#orchestrator-probes-ops-0102).

## Quick example

```php
use Waffle\Commons\Cache\Adapter\ArrayCache;
use Waffle\Commons\Cache\Pool\CachePool;
use Waffle\Commons\Cache\Factory\CacheFactory;
use Waffle\Commons\Contracts\Cache\Constant;

// PSR-16 directly
$psr16 = new ArrayCache(defaultTtl: 60);
$psr16->set('user:42', ['name' => 'Ada']);

// PSR-6 view over the same adapter
$pool = new CachePool($psr16);
$item = $pool->getItem('user:42');

// Factory wiring (the framework's preferred path)
$factory = new CacheFactory();
$redis = $factory->create(Constant::BACKEND_REDIS, [
    'dsn'         => 'redis://cache.internal:6379',
    'prefix'      => 'app',
    'default_ttl' => 3600,
]);
```
