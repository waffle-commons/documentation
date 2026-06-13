# Contracts Reference (`waffle-commons/contracts`)

> **Release:** `0.1.0-beta4`

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
    public function boot(): static;
    public function configure(): void;
    public function handle(ServerRequestInterface $request): ResponseInterface;
    public function reset(): void;
}
```

`reset()` is called between FrankenPHP worker requests to clear request-scoped state without re-booting.

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
| `Waffle\Commons\Contracts\Data\Connection\ConnectionPoolInterface` | Worker-safe pool of reusable PDO connections: `acquire(): PDO` (ping-before-dispense, transparent reconnect) and `release(PDO): void`. Implementations also implement `ResettableInterface`. |
| `Waffle\Commons\Contracts\Data\Migration\MigrationRunnerInterface` | Forward-only SQL migration runner: `run(?Closure $onApplied = null): list<string>` — applies pending migrations in version order, idempotently. |
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

Extends `Psr\EventDispatcher\ListenerProviderInterface`.

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

### `Waffle\Commons\Contracts\Http\ServerRequestFactoryInterface`

Framework-level factory for building a `ServerRequestInterface` from PHP superglobals (used by `WaffleRuntime`).

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

Building blocks for the ABAC ladder (`Level1Rule`…`Level10Rule`) and attribute-based voters.

### Attributes

- `Waffle\Commons\Contracts\Security\Attribute\Rule` — declares the required security level on a controller method or class.
- `Waffle\Commons\Contracts\Security\Attribute\Voter` — registers a class as an ABAC voter.
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
