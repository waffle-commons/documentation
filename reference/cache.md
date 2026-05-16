# Cache Reference (`waffle-commons/cache`)

> **Release:** `v0.1.0-beta0`
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

## Key validation

`Waffle\Commons\Cache\KeyValidator` enforces the PSR-16 key character set. Invalid keys throw `Waffle\Commons\Cache\Exception\InvalidCacheKeyException`.

## Stampede protection

`Waffle\Commons\Cache\Trait\StampedeAwareTrait` implements probabilistic early-expiration ("XFetch") to prevent thundering-herd misses under high concurrent load. Adapters that wish to opt in declare `implements StampedeProtectionInterface` and `use StampedeAwareTrait;`.

## Exceptions

All inherit from `Waffle\Commons\Cache\Exception\CacheException` and implement `Waffle\Commons\Contracts\Cache\Exception\CacheExceptionInterface`:

- `CacheBackendUnavailableException` — implements `CacheBackendUnavailableExceptionInterface`. Raised when an adapter's backing store cannot be reached.
- `InvalidCacheKeyException` — implements `InvalidCacheKeyExceptionInterface` + PSR-16's `Psr\SimpleCache\InvalidArgumentException`. Raised by `KeyValidator`.

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
