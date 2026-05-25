# Contracts Reference (`waffle-commons/contracts`)

> **Release:** `v0.1.0-beta1`

`waffle-commons/contracts` is the root package of the Waffle ecosystem. Every other component depends only on the contracts package plus its own declared PSR dependencies. The package contains interfaces, marker attributes, enums, exception interfaces, and ecosystem-wide typed constants — **no business logic**.

## Beta-1 changelog (one breaking change, three additions)

- **BREAKING** — `Waffle\Commons\Contracts\Security\Csrf\CsrfTokenManagerInterface::issue()`, `validate()`, and `refresh()` now take a `$sessionId` argument, folded into the HMAC payload so tokens bind to a single browser. See [security.md](security.md) and the [CSRF explanation](../explanation/security-csrf-double-submit.md).
- **NEW** — `Waffle\Commons\Contracts\Security\Attribute\PublicAccess`, the explicit opt-out for the new fail-closed ABAC default. See [attributes-public-access.md](attributes-public-access.md).
- **NEW** — `Waffle\Commons\Contracts\Routing\Exception\RouteNotFoundException` (concrete, `final`, implements the existing `RouteNotFoundExceptionInterface`). Thrown by `CoreRoutingMiddleware` so missing routes render as `404`, not `500`.
- **NEW** — CSRF binding constants on `Waffle\Commons\Contracts\Security\Csrf\Constant`: `SESSION_COOKIE_NAME = 'WAFFLE_SID'`, `SESSION_ID_BYTES = 32`, `SESSION_REQUEST_ATTRIBUTE = '_anon_sid'`, `SESSION_COOKIE_MAX_AGE = 2_592_000`, plus `MIN_SECRET_BYTES = 32` and `SECRET_ENV_KEY = 'WAFFLE_CSRF_SECRET'`.

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

- `Waffle\Commons\Contracts\Console\Enum\ExitCode` — `SUCCESS = 0`, `FAILURE = 1`, `USAGE = 2`, …
- `Waffle\Commons\Contracts\Console\Enum\Verbosity` — `QUIET`, `NORMAL`, `VERBOSE`, `VERY_VERBOSE`, `DEBUG`.

## Runtime

### `Waffle\Commons\Contracts\Runtime\RuntimeInterface`

The contract `WaffleRuntime` implements. Its single non-constructor method is:

```php
public function loop(KernelInterface $kernel, int $maxRequests = 500): void;
```

## System / Service

### `Waffle\Commons\Contracts\Service\ResettableInterface`

```php
interface ResettableInterface
{
    public function reset(): void;
}
```

Implemented by anything that needs per-request reset under FrankenPHP worker mode (Container, internal Kernel state, custom services).

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

- `Waffle\Commons\Contracts\Exception\Validation\ValidationExceptionInterface` — exposes `getField(): ?string` for RFC 7807 enrichment.
- `Waffle\Commons\Contracts\Routing\Exception\RouteNotFoundExceptionInterface` — and, since Beta-1, the concrete `RouteNotFoundException` (also in contracts) that the pipeline throws on a missed route.
- `Waffle\Commons\Contracts\Cache\Exception\CacheExceptionInterface` and friends.
- `Waffle\Commons\Contracts\Security\Exception\SecurityExceptionInterface` and friends.
- `Waffle\Commons\Contracts\Config\Exception\InvalidConfigurationExceptionInterface`.
- `Waffle\Commons\Contracts\Console\Exception\ConsoleExceptionInterface` (parent of `CommandNotFoundExceptionInterface`, `InvalidArgumentExceptionInterface`).

This interface-only approach lets the `ErrorHandlerMiddleware`'s status-code resolver match by interface and pick the right HTTP status without coupling to a specific implementation class.
