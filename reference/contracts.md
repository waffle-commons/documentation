# Contracts Reference (`waffle-commons/contracts`)

> **Release:** `0.1.0-beta6`

`waffle-commons/contracts` is the root package of the Waffle ecosystem. Every other component depends only on the contracts package plus its own declared PSR dependencies. The package contains interfaces, marker attributes, enums, exception interfaces, and ecosystem-wide typed constants — **no business logic**.

## Beta-1 changelog (one breaking change, multiple additions)

- **BREAKING** — `Waffle\Commons\Contracts\Security\Csrf\CsrfTokenManagerInterface::issue()`, `validate()`, and `refresh()` now take a `$sessionId` argument, folded into the HMAC payload so tokens bind to a single browser. See [security.md](security.md) and the [CSRF explanation](../explanation/security-csrf-double-submit.md).
- **NEW** — `Waffle\Commons\Contracts\Security\Attribute\PublicAccess`, the explicit opt-out for the new fail-closed ABAC default. See [attributes-public-access.md](attributes-public-access.md).
- **NEW** — `Waffle\Commons\Contracts\Routing\Exception\RouteNotFoundException` (concrete, `final`, implements the existing `RouteNotFoundExceptionInterface`). Thrown by `CoreRoutingMiddleware` so missing routes render as `404`, not `500`.
- **NEW** — CSRF binding constants on `Waffle\Commons\Contracts\Security\Csrf\Constant`: `SESSION_COOKIE_NAME = 'WAFFLE_SID'`, `SESSION_ID_BYTES = 32`, `SESSION_REQUEST_ATTRIBUTE = '_anon_sid'`, `SESSION_COOKIE_MAX_AGE = 2_592_000`, plus `MIN_SECRET_BYTES = 32` and `SECRET_ENV_KEY = 'WAFFLE_CSRF_SECRET'`.
- **NEW** — `Waffle\Commons\Contracts\Exception\WaffleExceptionInterface` base framework exception interface.
- **NEW** — `Waffle\Commons\Contracts\Routing\Exception\MethodNotAllowedExceptionInterface` and `MethodNotAllowedException` supporting HTTP 405 Method Not Allowed exceptions.
- **NEW** — `Waffle\Commons\Contracts\Routing\Attribute\Route` attribute migrated from the routing component to `contracts` with multi-method array support (`public array $methods = ['GET']`).
- **NEW** — HTTP method routing constants on `Waffle\Commons\Contracts\Routing\Constant`: `METHOD_GET = 'GET'`, `METHOD_POST = 'POST'`, etc.

## Core

### `Waffle\Commons\Contracts\Core\KernelInterface`

```php
namespace Waffle\Commons\Contracts\Core;

interface KernelInterface
{
    public ContainerInterface $container { get; }

    public function boot(): static;
    public function configure(): void;
    public function handle(ServerRequestInterface $request): ResponseInterface;
    public function reset(): void;
}
```

`reset()` is called between FrankenPHP worker requests to clear request-scoped state without re-booting.

*`$container` added in `0.1.0-beta7` (CARRY-04).* It is an interface property (PHP 8.4+): `{ get; }` requires every kernel to expose a publicly readable `$container`, and `waffle`'s `AbstractKernel` satisfies it with its promoted `public protected(set) ContainerInterface $container`. A test bridge such as `waffle-commons/testing` reads it to swap in test doubles between `boot()` and `configure()`, before the container locks. Application code keeps receiving its dependencies through constructors.

### `Waffle\Commons\Contracts\Core\TerminableInterface`

*Added in Beta-3.* A kernel capable of running work **after** the response has been emitted:

```php
public function terminate(ServerRequestInterface $request, ResponseInterface $response): void;
```

The Runtime invokes `terminate()` exactly once per request — after `emit()`, before `reset()` — so heavy post-response tasks (async dispatch, buffer flushing) never delay delivery. Support is optional: the Runtime guards the call with `instanceof`, so a kernel that does not implement the interface simply skips the post-response phase. Implemented by `waffle`'s `AbstractKernel`.

## Configuration

### `Waffle\Commons\Contracts\Config\ConfigInterface`

```php
namespace Waffle\Commons\Contracts\Config;

interface ConfigInterface
{
    public function getInt(string $key, ?int $default = null): ?int;
    public function getString(string $key, ?string $default = null): ?string;
    public function getArray(string $key, ?array $default = null): ?array;
    public function getBool(string $key, ?bool $default = null): ?bool;
}
```

Keys are dot-paths (e.g. `waffle.trusted_hosts`). All getters return `null` if the key is absent and no default is given. Mismatched-type values throw `Waffle\Commons\Contracts\Config\Exception\InvalidConfigurationExceptionInterface`.

## Container

### `Waffle\Commons\Contracts\Container\ContainerInterface`

```php
namespace Waffle\Commons\Contracts\Container;

use Psr\Container\ContainerInterface as PsrContainerInterface;
use Waffle\Commons\Contracts\Service\ResettableInterface;

interface ContainerInterface extends PsrContainerInterface, ResettableInterface
{
    public function get(string $id): mixed;
    public function has(string $id): bool;
    public function set(string $id, object|callable|string $concrete): void;
}
```

`ResettableInterface` adds `reset(): void` — the kernel calls it between requests so the instance cache is cleared while definitions stay registered.

### `Waffle\Commons\Contracts\Container\CompiledContainerInterface`

*Added in Beta-5 (RFC-019 / AOT).* Marker contract — `extends ContainerInterface` with no extra methods — identifying an ahead-of-time compiled container so the boot path can distinguish it from the reflection-based one. See [aot.md](aot.md).

## Routing

### `Waffle\Commons\Contracts\Routing\RouterInterface`

```php
namespace Waffle\Commons\Contracts\Routing;

interface RouterInterface
{
    public function boot(ContainerInterface $container): static;

    public function matchRequest(ServerRequestInterface $request): ?MatchedRoute;

    /** @return list<MatchedRoute> */
    public function getRoutes(): array;
}

final readonly class MatchedRoute
{
    public function __construct(
        public string $className,
        public string $method,
        public array  $arguments,
        public string $path,
        public string $name,
        public array  $params   = [],
        public int    $priority = 0,
        public array  $methods  = [],
    ) {}

    public function withParams(array $params): self;
}
```

`MatchedRoute` replaces the previous nested-array return shape — consumers now access typed properties (`$match->className`, `$match->params['id']`) instead of array keys. The interface still uses `?MatchedRoute` (null on miss) so `CoreRoutingMiddleware` can throw the typed `RouteNotFoundException`. Beta-1 Phase 3 adds `priority` — see [`reference/routing` → Priority & catch-all routes](routing.md#priority--catch-all-routes).

## Pipeline (PSR-15)

### `Waffle\Commons\Contracts\Pipeline\MiddlewareStackInterface`

The middleware-stack contract the kernel consumes. Implementations expose `add()` / `prepend()` builders and a `createHandler()` factory that returns the terminal PSR-15 `RequestHandlerInterface`.

## Cache (PSR-6 + PSR-16)

### `Waffle\Commons\Contracts\Cache\CacheInterface`

Extends `Psr\SimpleCache\CacheInterface` (PSR-16) so any PSR-16 method (`get`, `set`, `delete`, `has`, etc.) works against a Waffle cache adapter.

### `Waffle\Commons\Contracts\Cache\CacheItemPoolInterface`

Extends `Psr\Cache\CacheItemPoolInterface` (PSR-6) for clients that need the item-object API.

### `Waffle\Commons\Contracts\Cache\StampedeProtectionInterface`

Marker interface for adapters that implement probabilistic early-expiration ("stampede protection").

## Data (persistence, RFC-022)

Added in Beta-3; consumed by the `waffle-commons/data` component.

| Interface | Purpose |
| :--- | :--- |
| `Waffle\Commons\Contracts\Data\Connection\ConnectionPoolInterface` | Worker-safe pool of reusable connections, generalised in Beta-5: `acquire(): ConnectionInterface` (ping-before-dispense, transparent reconnect) and `release(ConnectionInterface): void`. Backend-typed specialisations narrow the surface — `RelationalConnectionPoolInterface` (`acquire(): PdoConnectionInterface`, plus `beginRequestScope()`/`endRequestScope()`) and `RedisConnectionPoolInterface`. Implementations also implement `ResettableInterface`. |
| `Waffle\Commons\Contracts\Data\Migration\MigrationRunnerInterface` | SQL migration runner: `run(?Closure $onApplied = null): list<string>` — applies pending migrations in version order, idempotently. *Added in `0.1.0-beta7` (OPS-03):* `rollback(int $steps = 1, ?Closure $onReverted = null): list<string>` reverts the most recent versions through their `V.down.sql` scripts, checking every target first; `status(): list<MigrationState>` reports each version. |
| `Waffle\Commons\Contracts\Data\Migration\MigrationState` | *Added in `0.1.0-beta7` (OPS-03).* `final readonly class` — one status row: `public string $version`, `public MigrationStatus $status`, `public bool $reversible`, `public ?string $appliedAt = null` (the ledger value as the driver returns it). |
| `Waffle\Commons\Contracts\Data\Migration\Enum\MigrationStatus` | *Added in `0.1.0-beta7` (OPS-03).* Backed enum: `Applied = 'applied'`, `Pending = 'pending'`, `Missing = 'missing'` (logged, but its script is no longer on disk). |
| `Waffle\Commons\Contracts\Data\Exception\DatabaseExceptionInterface` | Base contract for data-layer failures (`extends Throwable`); `getSqlState(): ?string` exposes the ANSI SQLSTATE when the backend provides one. |
| `Waffle\Commons\Contracts\Data\Exception\SecurityPathViolationExceptionInterface` | `extends DatabaseExceptionInterface` — a Firestore operation would target a non-isolated path (guardrail Rule 1). |
| `Waffle\Commons\Contracts\Data\Exception\UnauthenticatedAccessExceptionInterface` | `extends DatabaseExceptionInterface` — a guarded Firestore read/write was attempted without an authenticated identity (guardrail Rule 3). |
| `Waffle\Commons\Contracts\Data\Repository\RepositoryInterface` | The Stateless Repository Layer (RFC-022 §3), `@template T of object`: `find(QueryInterface): list<T>`, `findOne(QueryInterface): T|null`, `stream(QueryInterface): Generator<int, T>` (the §4.1 buffer-streaming path). No Active Record, no Unit of Work, no Identity Map. |
| `Waffle\Commons\Contracts\Data\Repository\WritableRepositoryInterface` | `extends RepositoryInterface` — the CRUD write surface: `save(object): void` (INSERT on null identity, UPDATE/upsert otherwise), `delete(object): void`, `findById(int\|string): ?T`. Implemented by all seven `waffle-commons/data` repositories. |
| `Waffle\Commons\Contracts\Data\Mapper\DataMapperInterface` | Pure Data Mapper (`@template T of object`) between an immutable entity and its flat storage row: `target()`, `identityField()`, `fields(): list<string>`, `identify(T): int\|string\|null`, `toRow(T): array<string, scalar\|null>`. Keeps entities free of persistence logic (no Active Record). |
| `Waffle\Commons\Contracts\Data\Warmup\DataWarmerInterface` | *Added in Beta-3.* CLI-side artifact warmer behind `bin/waffle data:warmup`: `warmUp(): list<string>` pre-compiles expensive artifacts (SQR trees, routing tables) into PHP cache files primed into OPcache shared memory. Implementations must be stateless and idempotent — warming never runs during an HTTP request. |

### The SQR vocabulary (`Contracts\Data\Query` + `Contracts\Data\Enum`)

The Semantic Query Representation (RFC-022 §3.1) is the shared language between repositories and every driver compiler. The read-side contracts are **PHP 8.4+ interface properties** (`public string $field { get; }`) — satisfied by `readonly` / `private(set)` implementations, no legacy getters:

| Contract | Surface |
| :--- | :--- |
| `Waffle\Commons\Contracts\Data\Query\QueryInterface` | `array $fields { get; }`, `?string $from { get; }`, `array $criteria { get; }`, `array $orderings { get; }`, `?int $limit { get; }`, `?int $offset { get; }` — immutable, compiler-agnostic, I/O-free. |
| `Waffle\Commons\Contracts\Data\Query\ComparisonInterface` | `string $field { get; }`, `Operator $operator { get; }`, `array $values { get; }` (always a list — set and scalar operators share one shape; values are bound, never interpolated). |
| `Waffle\Commons\Contracts\Data\Query\OrderInterface` | `string $field { get; }`, `Direction $direction { get; }`. |
| `Waffle\Commons\Contracts\Data\Enum\Operator: string` | `Equal '='` … `Like 'LIKE'` (nine cases); `isSetOperator(): bool`. **Relocated in Beta-3** from `Waffle\Commons\Data\Query\Operator` (⚠️ BC: update imports). |
| `Waffle\Commons\Contracts\Data\Enum\Direction: string` | `Ascending 'ASC'`, `Descending 'DESC'`. **Relocated in Beta-3** from `Waffle\Commons\Data\Query\Direction` (⚠️ BC). |

See [data.md](data.md) for the concrete implementations.

## Event dispatching (PSR-14)

### `Waffle\Commons\Contracts\EventDispatcher\EventDispatcherInterface`

Extends `Psr\EventDispatcher\EventDispatcherInterface`.

### `Waffle\Commons\Contracts\EventDispatcher\ListenerProviderInterface`

Standalone (does **not** extend the PSR interface): `getListenersForEvent(object $event): iterable<callable>`.

## Logging (PSR-3)

The framework uses `Psr\Log\LoggerInterface` directly. The reference implementation `Waffle\Commons\Log\StreamLogger` shows the framework's PHP 8.5 style:

```php
public function __construct(
    private(set) readonly string $streamPath = 'php://stderr',
    private(set) readonly string $channel = LogChannel::APP,
    private(set) readonly int $permissions = 0o644,
)
```

## HTTP

The framework relies on PSR-7 (`Psr\Http\Message\*`) and PSR-17 (`Psr\Http\Message\*FactoryInterface`). Beyond those, contracts adds:

### `Waffle\Commons\Contracts\Http\ResponseEmitterInterface`

```php
interface ResponseEmitterInterface
{
    public function emit(ResponseInterface $response): void;
}
```

### `Waffle\Commons\Contracts\Http\GlobalsFactoryInterface`

```php
interface GlobalsFactoryInterface
{
    public function createFromGlobals(): ServerRequestInterface;
}
```

Builds a PSR-7 `ServerRequestInterface` from the ambient SAPI/superglobals. `WaffleRuntime` depends on this port — **not** `http`'s concrete `GlobalsFactory` — so the runtime stays within the contracts-only perimeter; the app injects the concrete.

### `Waffle\Commons\Contracts\Http\ServerRequestFactoryInterface`

Extends PSR-17's `ServerRequestFactoryInterface` (`createServerRequest(string $method, $uri)`) — the explicit-arguments request factory, distinct from the from-globals `GlobalsFactoryInterface` above.

### `Waffle\Commons\Contracts\Http\ClientIpResolverInterface`

*Added in `0.1.0-beta7` (NET-01).*

```php
interface ClientIpResolverInterface
{
    /** The client's IP address, or null when the request carries none (a request built outside a server). */
    public function resolve(ServerRequestInterface $request): ?string;
}
```

Behind a proxy `REMOTE_ADDR` is the proxy's address, and `X-Forwarded-For` holds whatever the client wrote
unless a trusted proxy appended to it. An implementation trusts the forwarded chain only from an explicit
allow-list of proxies and returns the right-most address that is not one, so a client cannot choose its own
key. Every IP-bound check goes through it. Implementation: `http`'s `TrustedProxyClientIpResolver`
([http.md](http.md#client-ip-behind-trusted-proxies-net-01)).

## Rate limiting (NET-01)

*Added in `0.1.0-beta7` (NET-01, RFC-017 §3.1, RFC-024 §5.1).* Implementations: `security`'s `RateLimiter`
and `RateLimitMiddleware` ([security.md](security.md#rate-limiting-net-01)), `data`'s
`RedisLimiterStateStorage` ([data.md](data.md#shared-state-for-limiters-and-breakers-net-0103)).

The value objects in this section and the next validate every write with a **property hook**. `TokenBucket`
and `TimeoutPolicy` are shown in full; elsewhere a hook body written `{ /* ≥ 1 */ }` is elided, and every
elided hook has the same shape — `if (<outside the bound>) { throw new InvalidArgumentException('…'); }`
then `$this->property = $value;`.

### `Waffle\Commons\Contracts\RateLimit\RateLimiterInterface`

```php
interface RateLimiterInterface
{
    /**
     * @param string $key  The bucket's key, e.g. "ip:203.0.113.7" or "login:alice"; never empty.
     * @param int    $cost Between 1 and the bucket capacity.
     *
     * @throws InvalidArgumentException When the key is empty or the cost does not fit the bucket.
     */
    public function consume(string $key, int $cost = 1): RateLimitDecision;
}
```

A storage outage never surfaces as an exception: the limiter applies its failure mode and says so in the
decision (`$enforced`).

### `Waffle\Commons\Contracts\RateLimit\LimiterStateStorageInterface`

```php
interface LimiterStateStorageInterface
{
    /**
     * @throws InvalidArgumentException       When the key is empty or the cost does not fit the bucket.
     * @throws StateStorageExceptionInterface When the storage is unreachable or refuses the operation.
     */
    public function consume(string $key, TokenBucket $bucket, int $cost): RateLimitDecision;
}
```

Atomically refills the bucket of `$key` for the time elapsed since its last use, then takes `$cost` tokens
if it holds that many; a new bucket starts full. Each operation is atomic, and the storage keeps the time
itself, so concurrent workers — and hosts whose clocks drift — agree on every bucket. An in-memory
implementation is correct for a single worker only.

### `Waffle\Commons\Contracts\RateLimit\TokenBucket`

```php
final class TokenBucket
{
    /** The longest refill period: one day. */
    public const int MAX_PERIOD_SECONDS = 86_400;

    public function __construct(
        public private(set) int $capacity {
            set(int $value) {
                if ($value < 1) {
                    throw new InvalidArgumentException('A token bucket holds at least one token.');
                }
                $this->capacity = $value;
            }
        },
        public private(set) int $periodSeconds {
            set(int $value) {
                if ($value < 1 || $value > self::MAX_PERIOD_SECONDS) {
                    throw new InvalidArgumentException('A token bucket refills in 1 to 86400 seconds.');
                }
                $this->periodSeconds = $value;
            }
        },
    ) {}
}
```

Holds at most `$capacity` tokens and refills completely in `$periodSeconds`: "100 requests a minute" is
`new TokenBucket(100, 60)`, and the capacity is also the largest burst. The constructor-promoted properties
combine **asymmetric visibility** (`public private(set)`: readable everywhere, writable only inside the
class) with **property hooks** (`set(int $value) { … }`) that validate every write, so an invalid bucket
cannot exist.

### `Waffle\Commons\Contracts\RateLimit\RateLimitDecision`

```php
final class RateLimitDecision
{
    public function __construct(
        public private(set) bool $allowed,
        public private(set) int $limit { set(int $value) { /* ≥ 1 */ } },
        public private(set) int $remaining { set(int $value) { /* ≥ 0 */ } },
        public private(set) int $resetSeconds { set(int $value) { /* ≥ 0 */ } },
        public private(set) int $retryAfterSeconds = 0 { set(int $value) { /* ≥ 0 */ } },
        public private(set) bool $enforced = true,
    );
}
```

What the response headers need: `$limit` → `RateLimit-Limit`, `$remaining` → `RateLimit-Remaining`,
`$resetSeconds` → `RateLimit-Reset`, `$retryAfterSeconds` → `Retry-After` on a `429`. Each hook throws an
`InvalidArgumentException` on a value below its bound; the constructor also refuses `$remaining > $limit`,
and requires a denied decision to carry a `Retry-After` of at least 1 second and an allowed one none.
`$enforced` is false when the limiter could not reach its storage and applied its failure mode: the counts
are unknown, so the `RateLimit-*` headers are omitted.

### `Waffle\Commons\Contracts\RateLimit\Exception\StateStorageExceptionInterface`

`interface StateStorageExceptionInterface extends Throwable {}` — the shared state is unavailable. The
limiter and the breaker catch it to apply their failure mode without coupling to the storage.

## Outbound resilience (NET-02/03)

*Added in `0.1.0-beta7` (NET-02/03, RFC-024).* Implementations: `http-client`'s `Client`,
`RetryingClient` and `CircuitBreakerClient` ([http-client.md](http-client.md#resilience-net-0203)),
`data`'s `RedisCircuitStateStorage` ([data.md](data.md#shared-state-for-limiters-and-breakers-net-0103)).

### `Waffle\Commons\Contracts\HttpClient\TimeoutPolicy`

```php
final class TimeoutPolicy
{
    public const int DEFAULT_CONNECT_MS = 1000;
    public const int DEFAULT_TOTAL_MS = 10_000;
    public const int MAX_CONNECT_MS = 30_000;
    public const int MAX_TOTAL_MS = 120_000;

    public function __construct(
        public private(set) int $connectMs = self::DEFAULT_CONNECT_MS {
            set(int $value) {
                if ($value < 1 || $value > self::MAX_CONNECT_MS) {
                    throw new InvalidArgumentException('A connect timeout lies between 1 and 30000 ms.');
                }
                $this->connectMs = $value;
            }
        },
        public private(set) int $totalMs = self::DEFAULT_TOTAL_MS {
            set(int $value) {
                if ($value < 1 || $value > self::MAX_TOTAL_MS) {
                    throw new InvalidArgumentException('A total timeout lies between 1 and 120000 ms.');
                }
                $this->totalMs = $value;
            }
        },
    ) {}
}
```

No infinite timeout can be built: cURL reads `0` as "no timeout", so the hooks refuse `≤ 0` and anything
above the ceilings.

### `Waffle\Commons\Contracts\HttpClient\RetryPolicy`

```php
final class RetryPolicy
{
    /** @var list<int> */
    public const array RETRYABLE_STATUSES = [429, 502, 503, 504];

    public function __construct(
        public private(set) int $maxAttempts = 3 { set(int $value) { /* 1 to 10 */ } },
        public private(set) int $baseDelayMs = 100 { set(int $value) { /* ≥ 1 */ } },
        public private(set) int $maxDelayMs = 2000 { set(int $value) { /* ≥ 1 */ } },
        public private(set) int $budgetMs = 15_000 { set(int $value) { /* ≥ 1 */ } },
        public private(set) bool $retryNonIdempotent = false,
    ) {}
}
```

A transport failure or a retryable status is retried with exponential backoff and full jitter, up to
`$maxAttempts` attempts in all, and no wait starts once `$budgetMs` would be exceeded. Only idempotent
methods (RFC 9110: `GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE`) are retried unless `$retryNonIdempotent` is
set. A `Retry-After` may exceed `$maxDelayMs` when it fits the remaining budget (RFC-024 §5.4). Each hook
throws an `InvalidArgumentException` outside its bounds.

### `Waffle\Commons\Contracts\HttpClient\SettlingClientInterface`

```php
interface SettlingClientInterface extends ConcurrentClientInterface
{
    /**
     * @param array<array-key, RequestInterface> $requests
     *
     * @return array<array-key, ResponseInterface|ClientExceptionInterface> One outcome per request, in the
     *         batch's key order.
     */
    public function settleRequests(array $requests): array;
}
```

A concurrent client that settles a whole batch: every request ends with its response or the exception its
transfer failed with, and none is thrown. The resilience decorators need every outcome — the breaker counts
each, the retry policy resends only what failed — so a decorator below them implements this to keep a batch
concurrent (`auth`'s `AuthenticatedSettlingClient`, for one); a client that does not is sent one request at
a time.

### `Waffle\Commons\Contracts\CircuitBreaker\CircuitBreakerPolicy`

```php
final class CircuitBreakerPolicy
{
    public function __construct(
        public private(set) int $failureRatePercent = 50 { set(int $value) { /* 1 to 100 */ } },
        public private(set) int $minimumCalls = 10 { set(int $value) { /* ≥ 1 */ } },
        public private(set) int $windowSeconds = 60 { set(int $value) { /* 1 to 3600 */ } },
        public private(set) int $coolDownSeconds = 30 { set(int $value) { /* 1 to 3600 */ } },
        public private(set) int $halfOpenTrials = 1 { set(int $value) { /* ≥ 1 */ } },
    ) {}
}
```

Closed → open when, within `$windowSeconds`, at least `$minimumCalls` calls were recorded and
`$failureRatePercent` % or more failed. Open → half-open after `$coolDownSeconds`. Half-open admits
`$halfOpenTrials` trial calls: a success closes the circuit, a failure opens it again. Each hook throws an
`InvalidArgumentException` outside its bounds.

### `Waffle\Commons\Contracts\CircuitBreaker\CircuitStateStorageInterface`

```php
interface CircuitStateStorageInterface
{
    /** @throws StateStorageExceptionInterface When the storage is unreachable. */
    public function acquire(string $circuit, CircuitBreakerPolicy $policy): CircuitDecision;

    /** @throws StateStorageExceptionInterface When the storage is unreachable. */
    public function record(string $circuit, CircuitBreakerPolicy $policy, bool $success): CircuitDecision;
}
```

`acquire()` answers whether a call may go now — always when closed, never while open before the cool-down,
for at most `$halfOpenTrials` calls when half-open — and turns an open circuit whose cool-down elapsed
half-open. `record()` reports the outcome and opens or closes the circuit as the policy says. Every
operation is atomic, in the same storage family as the limiter's.

### `CircuitDecision`, `CircuitState`, `CircuitStateChanged`, `CircuitOpenExceptionInterface`

```php
final readonly class CircuitDecision
{
    public function __construct(
        public bool $allowed,
        public CircuitState $state,
        public ?CircuitState $previous = null,
    ) {}

    /** True when this operation moved the circuit from `$previous` to `$state`. */
    public function isTransition(): bool;
}

enum CircuitState: string
{
    case Closed = 'closed';
    case HalfOpen = 'half_open';
    case Open = 'open';
}

final readonly class CircuitStateChanged   // Contracts\CircuitBreaker\Event
{
    /** @param string $circuit The upstream origin the circuit protects, e.g. "https://api.example.com:443". */
    public function __construct(
        public string $circuit,
        public CircuitState $from,
        public CircuitState $to,
    ) {}
}

interface CircuitOpenExceptionInterface extends Throwable {}   // Contracts\CircuitBreaker\Exception
```

`CircuitStateChanged` is dispatched through PSR-14, so listeners in any component react without importing
the client that owns the breaker. A `CircuitOpenExceptionInterface` means the call failed fast before any
connection; its code is the HTTP status the error handler renders (`503`), and it is never retried.

## Health checks (OPS-01)

*Added in `0.1.0-beta7` (OPS-01, RFC-023 §3.1).* Runner: `pipeline`'s `ReadinessProbe`
([pipeline.md](pipeline.md#orchestrator-probes-ops-0102)).

### `Waffle\Commons\Contracts\Health\HealthCheckInterface`

```php
interface HealthCheckInterface
{
    /** Stable identifier, used in the detailed report and in `waffle.health.required`: "db.pool", "redis". */
    public function name(): string;

    /** Whether a failure of this probe makes the instance not ready. */
    public function isCritical(): bool;

    /** Probes the dependency within the probe's own timeout. Never throws: a failure is a down result. */
    public function check(): HealthResult;
}
```

Components ship their own probes; the application registers them and declares the required ones by name.
A probe stays cheap — it leases a pooled connection rather than opening one — because it runs on every
`/readyz` call.

### `Waffle\Commons\Contracts\Health\HealthResult` and `Enum\HealthStatus`

```php
final readonly class HealthResult
{
    private function __construct(
        public HealthStatus $status,
        public string $detail,
    ) {}

    public static function up(string $detail = ''): self;

    public static function down(string $detail): self;
}

enum HealthStatus: string
{
    case Up = 'up';
    case Down = 'down';
}
```

Binary on purpose (RFC-023 §5.4). `$detail` is meant for the private report only: the public body never
shows it, because dependency topology is reconnaissance data. The probe runner measures each duration
itself; a probe does not report it.

## Security

### `Waffle\Commons\Contracts\Security\SecurityInterface`

```php
namespace Waffle\Commons\Contracts\Security;

interface SecurityInterface
{
    /**
     * @param string[] $expectations
     * @throws SecurityExceptionInterface|\ReflectionException
     */
    public function analyze(object $object, array $expectations = []): void;
}
```

### `Waffle\Commons\Contracts\Security\SecurityRuleInterface` / `VoterInterface`

Building blocks for the ABAC ladder (`Level1Rule`…`Level10Rule`) and attribute-based voters. A voter implements `decide(SecurityContextInterface $ctx, mixed $subject = null): bool` — it receives the request-scoped security context (the authenticated identity — with its `roles` — and the client IP) and the subject under decision (the resolved resource, or the PSR-7 request), so ownership/IDOR rules are expressible.

### `Waffle\Commons\Contracts\Security\SubjectResolverInterface`

*Added in Beta-6 (SEC-05).* Resolves the domain subject under decision for the current request:

```php
public function resolve(ServerRequestInterface $request): mixed;
```

The security middleware invokes it post-routing / pre-dispatch (route parameters are already request attributes) and threads the returned value into every voter as `$subject`, taking precedence over the bare request. Fail-closed: a resolution failure MUST throw (the caller denies the request); `null` is reserved for routes that genuinely carry no resource. Implementations must be stateless.

### Attributes

- `Waffle\Commons\Contracts\Security\Attribute\PublicAccess` — method-only opt-out from fail-closed ABAC (see [attributes-public-access.md](attributes-public-access.md)).
- `Waffle\Commons\Contracts\Security\Attribute\Voter` — declares a required ABAC voter (by FQCN) on a controller class or method.
- `Waffle\Commons\Contracts\Security\Csrf\Attribute\RequiresCsrfToken` — marks a route as requiring CSRF validation.

## Handlers

### `Waffle\Commons\Contracts\Handler\ArgumentResolverInterface`

Resolves controller method arguments — including `#[Dto]`-marked DTOs hydrated from the parsed request body.

### `Waffle\Commons\Contracts\Handler\ResponseConverterInterface`

Converts a controller's scalar/array return into a PSR-7 `ResponseInterface`.

## Validation

### `Waffle\Commons\Contracts\Validation\ValidatorInterface`

```php
interface ValidatorInterface
{
    public function validate(object $value, array $groups = []): ValidationResultInterface;
}
```

### `ValidationResultInterface`

```php
interface ValidationResultInterface
{
    public function isValid(): bool;

    /** @return ViolationInterface[] */
    public function getViolations(): array;
}
```

### `ViolationInterface`

```php
interface ViolationInterface
{
    public function getMessage(): string;
    public function getPropertyPath(): string;
    public function getInvalidValue(): mixed;
}
```

Beta-1 strongly favors **Property Hooks on `readonly` DTOs** over external validators — the validator interfaces are reserved for cases where validation must happen outside the DTO's own constructor.

## Marker attributes

### `Waffle\Commons\Contracts\Attribute\Dto`

```php
#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Dto {}
```

`ControllerArgumentResolver` detects this attribute on a controller parameter's type, decodes the PSR-7 parsed body, and instantiates the DTO. Validation happens inside the DTO's PHP 8.5 Property Hooks.

## Console

| Interface | Purpose |
| :--- | :--- |
| `Waffle\Commons\Contracts\Console\ConsoleApplicationInterface` | The CLI runtime contract (`add`, `find`, `has`, `all`, `run`). |
| `Waffle\Commons\Contracts\Console\CommandInterface` | A single command (`getName`, `getDescription`, `execute(InputInterface, OutputInterface)`). |
| `Waffle\Commons\Contracts\Console\InputInterface` | CLI input abstraction (positional + named options). |
| `Waffle\Commons\Contracts\Console\OutputInterface` | CLI output abstraction (`writeLine`, `writeError`, `setVerbosity`). |
| `Waffle\Commons\Contracts\Console\Constant` | Typed constants (`DEFAULT_APP_NAME`, `COMMAND_LIST`). |

Plus enums:

- `Waffle\Commons\Contracts\Console\Enum\ExitCode` — `SUCCESS = 0`, `FAILURE = 1`, `USAGE = 64`, `DATA_ERR = 65`, `NO_INPUT = 66`, `NO_PERM = 77`, `CONFIG = 78` (BSD `sysexits(3)`); plus `isSuccess()` / `isFailure()`.
- `Waffle\Commons\Contracts\Console\Enum\Verbosity` — `QUIET`, `NORMAL`, `VERBOSE`, `VERY_VERBOSE`, `DEBUG`.

## Runtime

### `Waffle\Commons\Contracts\Runtime\RuntimeInterface`

The contract `WaffleRuntime` implements. Its single non-constructor method is:

```php
public function loop(KernelInterface $kernel, int $maxRequests = 500): void;
```

### `Waffle\Commons\Contracts\Runtime\AuditRunnerInterface`

Contract for running an external audit script and streaming its output (consumed by the `igor:audit` console command; concrete `ProcessAuditRunner` lives in `waffle-commons/runtime`). Executes the script without a shell and reports each line to a `Closure`:

```php
public function run(string $scriptPath, string $workingDirectory, array $arguments, Closure $onLine): int;
```

## System / Service

### `Waffle\Commons\Contracts\Service\ResettableInterface`

```php
interface ResettableInterface
{
    public function reset(): void;
}
```

Implemented by anything that needs per-request reset under FrankenPHP worker mode (Container, internal Kernel state, the data layer's `PDOConnectionPool`, custom services).

### `Waffle\Commons\Contracts\System\SystemInterface`

Internal binding between `KernelInterface` and `SecurityInterface`; consumed by `Waffle\Core\System`.

## Constants

### `Waffle\Commons\Contracts\Constant\Constant`

`final class Constant` is the canonical typed-constant store used across the ecosystem:

```php
public const string APP_ENV          = 'APP_ENV';
public const string APP_DEBUG        = 'APP_DEBUG';
public const string ENV_PROD         = 'prod';
public const string ENV_DEV          = 'dev';
public const string ENV_TEST         = 'test';
public const string ENV_STAGING      = 'staging';
public const string ENV_EXCEPTION    = 'exception';
public const string FAILSAFE_DEFAULT = self::DISABLED;
public const string ENABLED          = 'enabled';
public const string DISABLED         = 'disabled';
public const int    SECURITY_LEVEL0  = 0;
// … SECURITY_LEVEL1 … SECURITY_LEVEL10
public const string TYPE_STRING      = 'string';
public const string TYPE_INT         = 'int';
public const string TYPE_VOID        = 'void';
public const string TYPE_MIXED       = 'mixed';
public const string METHOD_GET       = 'GET';
public const string METHOD_POST      = 'POST';
public const string ATTR_CLASSNAME   = '_classname';
public const string ATTR_CONTROLLER  = '_controller';
public const string ATTR_METHOD      = '_method';
```

Component-specific constant classes live alongside their interfaces: `Waffle\Commons\Contracts\Cache\Constant` (backend identifiers `BACKEND_ARRAY`, `BACKEND_FILE`, `BACKEND_REDIS`), `Waffle\Commons\Contracts\Console\Constant`, `Waffle\Commons\Contracts\Security\Csrf\Constant`, etc.

## Enums

- `Waffle\Commons\Contracts\Enum\Failsafe` — backed-string enum (`ENABLED`/`DISABLED`) governing the Config component's failsafe boot path.
- `Waffle\Commons\Contracts\Console\Enum\ExitCode` — typed CLI exit codes.
- `Waffle\Commons\Contracts\Console\Enum\Verbosity` — verbosity ladder.

## Exception interfaces

The contracts package declares interfaces (never concrete exceptions). Implementations live in each consuming component.

- `Waffle\Commons\Contracts\Exception\WaffleExceptionInterface` — the base framework exception interface.
- `Waffle\Commons\Contracts\Exception\Validation\ValidationExceptionInterface` — exposes `getField(): ?string` for RFC 7807 enrichment.
- `Waffle\Commons\Contracts\Routing\Exception\RouteNotFoundExceptionInterface` — and, since Beta-1, the concrete `RouteNotFoundException` (also in contracts) that the pipeline throws on a missed route.
- `Waffle\Commons\Contracts\Routing\Exception\MethodNotAllowedExceptionInterface` — and the concrete `MethodNotAllowedException` thrown on 405 Method Not Allowed errors, exposing `getAllowedMethods()`.
- `Waffle\Commons\Contracts\Cache\Exception\CacheExceptionInterface` and friends.
- `Waffle\Commons\Contracts\Security\Exception\SecurityExceptionInterface` and friends.
- `Waffle\Commons\Contracts\Config\Exception\InvalidConfigurationExceptionInterface`.
- `Waffle\Commons\Contracts\Console\Exception\ConsoleExceptionInterface` (parent of `CommandNotFoundExceptionInterface`, `InvalidArgumentExceptionInterface`).

This interface-only approach lets the `ErrorHandlerMiddleware`'s status-code resolver match by interface and pick the right HTTP status without coupling to a specific implementation class.
