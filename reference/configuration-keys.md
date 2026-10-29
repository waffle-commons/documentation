# Configuration Keys Reference

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *Every `waffle.*` key of the skeleton's `config/app.yaml`, with its reader*

Each key below is declared in the skeleton's `config/app.yaml`. **Template value** is what that file sets;
**default** is what the code uses when the key is absent. A `%env(VAR)%` value is read from the environment
registry (the process environment wins over `.env`, see [config.md](config.md#environment-registry)).
The keys are frozen by beta8 `[FRZ-03]`; names may still change until then. `waffle.debug`, `waffle.log.channel` and
`waffle.log.stream` were removed in `0.1.0-beta7`: nothing read them (`public/index.php` reads `APP_DEBUG` itself,
and the kernel factory builds its loggers with their channels).

Most keys are read by the application's kernel factory (`App\Factory\*` in the skeleton), not by the
components: a component receives the value through its constructor. Where a component reads a key itself,
the reader names it.

## Runtime

| Key | Type | Template value | Default | Read by |
| :--- | :--- | :--- | :--- | :--- |
| `waffle.env` | string | `%env(APP_ENV)%` | — | The demo `AuthDemoController` only. The kernel's environment comes from `APP_ENV` in `public/index.php`. |
| `waffle.trusted_hosts` | list<string> | `[localhost]` | `[]` | `AppKernelFactory` → `TrustedHostMiddleware`. The `TRUSTED_HOSTS` environment variable (comma-separated) replaces the list when it is set. |

## Probes and shutdown — OPS-01/02

| Key | Type | Template value | Default | Read by |
| :--- | :--- | :--- | :--- | :--- |
| `waffle.health.required` | list<string> | `[db.pool]` | `[]` | `AppKernelFactory` → `ReadinessProbe`. A listed probe is critical; a listed name with no registered probe makes the pod not ready. |
| `waffle.health.drain_marker` | string | `/tmp/waffle-draining` | `null` (no marker) | `AppKernelFactory` → `ReadinessProbe`. While the file exists, `/readyz` answers 503. |

`CADDY_GRACE_PERIOD` (25 s) and `CADDY_SHUTDOWN_DELAY` (0 s) are environment variables of the Caddyfile,
not `waffle.*` keys — see [How to: Run on Kubernetes](../how-to/run-on-kubernetes.md#2-drain-before-sigterm).

## Security

| Key | Type | Template value | Default | Read by |
| :--- | :--- | :--- | :--- | :--- |
| `waffle.security.level` | int | `10` | `1` | `Waffle\Commons\Security\Security` itself. The `SecureContainer` integrity ladder, 1 to 10. |
| `waffle.security.csrf.secret` | string | `%env(WAFFLE_CSRF_SECRET)%` | the `WAFFLE_CSRF_SECRET` variable | `AppKernelFactory::resolveCsrfSecret()`. Shorter than 32 bytes in production: the boot is refused. |
| `waffle.security.cors.allowed_origins` | list<string> | `[]` | `[]` | `AppKernelFactory` → `CorsPolicy`. Empty refuses every cross-origin request. |
| `waffle.security.ssrf.allowed_hosts` | list<string> | `[]` | `[]` | `AppKernelFactory` → `SsrfGuard`. Hosts or CIDR ranges exempted from the private-address check. |

### Rate limiting — NET-01

Read by the skeleton's `App\Factory\RateLimitFactory`. See [How to: Rate-Limit Requests](../how-to/rate-limit-requests.md).

| Key | Type | Template value | Default | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `waffle.security.rate_limit.enabled` | bool | `true` | off | Anything but `true` mounts nothing. |
| `waffle.security.rate_limit.capacity` | int | `120` | `120` | Requests per period, bursts included (`TokenBucket::$capacity`, ≥ 1). |
| `waffle.security.rate_limit.period_seconds` | int | `60` | `60` | Full refill time, 1 to 86 400. |
| `waffle.security.rate_limit.on_storage_failure` | string | `open` | `open` | `open` or `closed` (`StorageFailureMode`); an unknown value means `open`. |
| `waffle.security.rate_limit.trusted_proxies` | list<string> | `[]` | `[]` | Proxy addresses or CIDR ranges whose `X-Forwarded-For` is read. A non-string entry is ignored; a malformed one refuses the boot. |
| `waffle.security.rate_limit.redis_dsn` | string | `%env(REDIS_DSN)%` | `redis://127.0.0.1:6379` | `redis://[:password@]host[:port]`. *Added `0.1.0-beta7`.* |

A Redis outage applies `on_storage_failure` with a warning and increments `waffle_ratelimit_storage_errors_total{mode}`.

## Outbound HTTP client — NET-02/03

Read by the skeleton's `App\Factory\HttpClientFactory`; a missing key takes the contract default. See
[How to: Make Outbound Calls Resilient](../how-to/make-outbound-calls-resilient.md).

| Key | Type | Template value | Default | Bounds |
| :--- | :--- | :--- | :--- | :--- |
| `waffle.http_client.resilience.connect_timeout_ms` | int | `1000` | `1000` | 1 to 30 000 (`TimeoutPolicy`) |
| `waffle.http_client.resilience.total_timeout_ms` | int | `10000` | `10000` | 1 to 120 000 |
| `waffle.http_client.resilience.max_attempts` | int | `3` | `3` | 1 to 10, the first attempt included (`RetryPolicy`) |
| `waffle.http_client.resilience.base_delay_ms` | int | `100` | `100` | ≥ 1 |
| `waffle.http_client.resilience.max_delay_ms` | int | `2000` | `2000` | ≥ 1; a `Retry-After` may exceed it within the budget |
| `waffle.http_client.resilience.budget_ms` | int | `15000` | `15000` | ≥ 1; no wait starts past it |
| `waffle.http_client.resilience.retry_non_idempotent` | bool | `false` | `false` | `true` also retries `POST` / `PATCH` |
| `waffle.http_client.breaker.enabled` | bool | `true` | off | Anything but `true` builds no breaker. |
| `waffle.http_client.breaker.failure_rate_percent` | int | `50` | `50` | 1 to 100 (`CircuitBreakerPolicy`) |
| `waffle.http_client.breaker.minimum_calls` | int | `10` | `10` | ≥ 1 |
| `waffle.http_client.breaker.window_seconds` | int | `60` | `60` | 1 to 3 600 |
| `waffle.http_client.breaker.cool_down_seconds` | int | `30` | `30` | 1 to 3 600 |
| `waffle.http_client.breaker.half_open_trials` | int | `1` | `1` | ≥ 1 |
| `waffle.http_client.breaker.redis_dsn` | string | `%env(REDIS_DSN)%` | `redis://127.0.0.1:6379` | *Added `0.1.0-beta7`.* |

## Universal Authentication Bridge — RFC-021

| Key | Type | Template value | Default | Read by |
| :--- | :--- | :--- | :--- | :--- |
| `waffle.auth.secret` | string | `%env(WAFFLE_AUTH_SECRET)%` | the `WAFFLE_AUTH_SECRET` variable | `AppKernelFactory::resolveAuthSecret()`. Shorter than 32 bytes in production: the boot is refused. |
| `waffle.auth.tenant` | string | `skeleton` | `null` | `AssertionCredentialsProvider` (`ten` claim of outbound assertions). |
| `waffle.auth.jwt.issuer` | string | `https://waffle-dev.local` | same | `JwtConfig` of the demo Bearer scheme. |
| `waffle.auth.jwt.audience` | string | `waffle-skeleton` | same | `JwtConfig`. |
| `waffle.auth.webauthn.rp_id` | string | `localhost` | same | `WebAuthnLibAdapter`; a domain, never an origin. |
| `waffle.auth.webauthn.rp_name` | string | `Waffle Skeleton` | same | `WebAuthnLibAdapter`. |
| `waffle.auth.webauthn.allowed_origins` | list<string> | `[https://localhost]` | same | `WebAuthnLibAdapter`. |
| `waffle.auth.assert_hosts` | list<string> | `[]` | `[]` | `AssertionCredentialsProvider`: hosts that receive the signed `X-Wfl-Assert-User` assertion. |

## Paths

| Key | Type | Template value | Default | Read by |
| :--- | :--- | :--- | :--- | :--- |
| `waffle.paths.controllers` | string | `src/Controller` | none | `AbstractKernel` (container discovery, under `APP_ROOT`), the `Router`, and `bin/waffle`. |
| `waffle.paths.services` | string | `src/Service` | none | `AbstractKernel` (container discovery, under `APP_ROOT`). |
| `waffle.paths.event_listeners` | string | `src/EventListener` | none | `EventListenerDiscovery`, under `APP_ROOT` like the two keys above. *Fixed in `0.1.0-beta7`:* the template pointed at `app/EventListener`, which the skeleton does not have, and the path was not resolved under `APP_ROOT`. |

## Cache — RFC-013

Read by `AppKernelFactory::buildCache()` → `Waffle\Commons\Cache\Factory\CacheFactory`.

| Key | Type | Template value | Default | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `waffle.cache.adapter` | string | `redis` | `array` | `array`, `file` or `redis`. `redis` also registers the cache readiness probe. |
| `waffle.cache.directory` | string | `var/cache/psr16` | `var/cache/psr16` | `file` adapter, under `APP_ROOT`. |
| `waffle.cache.redis_dsn` | string | `%env(REDIS_DSN)%` | Predis default (`tcp://127.0.0.1:6379`) | `redis` adapter, through Predis. |
| `waffle.cache.default_ttl` | int | `3600` | none | Seconds. |
| `waffle.cache.prefix` | string | `waffle:cache:` | none | `redis` adapter. |

With `adapter: redis`, the cache must reach Redis when the kernel boots: the router reads its route cache
there, and `RedisCache` throws `CacheBackendUnavailableException` rather than serving without it.

## Database — RFC-022

Read by the skeleton's `App\Factory\ConnectionPoolFactory`, and `migrations_path` by `MigrationRunner` too.
See [How to: Run Database Migrations](../how-to/database-migrations.md).

| Key | Type | Template value | Default |
| :--- | :--- | :--- | :--- |
| `waffle.database.driver` | string | `pgsql` | `pgsql` |
| `waffle.database.host` | string | `%env(DB_HOST)%` | `127.0.0.1` |
| `waffle.database.port` | string | `%env(DB_PORT)%` | `5432` |
| `waffle.database.database` | string | `%env(DB_NAME)%` | `''` |
| `waffle.database.username` | string | `%env(DB_USER)%` | `waffle` |
| `waffle.database.password` | string | `%env(DB_PASSWORD)%` | `''` |
| `waffle.database.charset` | string | `utf8` | `utf8` (ignored by `pgsql`) |
| `waffle.database.pool_size` | string | `%env(DB_POOL_SIZE)%` | `8`; an invalid value falls back to it |
| `waffle.database.ping_freshness_ms` | string | `%env(DB_PING_FRESHNESS_MS)%` | `500`; `0` probes every lease, a negative or invalid value falls back to `500` (*added in `0.1.0-beta7`*, CARRY-01) |
| `waffle.database.migrations_path` | string | `migrations` | `migrations` (`MigrationRunner::DEFAULT_PATH`) |

## Related

- [Config Reference](config.md) — the loader, the typed getters and the environment registry.
- [How to: Manage Configuration](../how-to/configuration.md) — adding a key and injecting it.
