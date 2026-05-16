# Container Reference (`waffle-commons/container`)

> **Release:** `v0.1.0-beta0`
> **PSR Compliance:** PSR-11 (`Psr\Container\ContainerInterface`)

A strict PSR-11 container with reflection-based autowiring, circular-dependency detection, and worker-mode reset semantics. Core services (the PSR-11 `ContainerInterface` itself) cannot be redefined after registration.

## Constructor

```php
namespace Waffle\Commons\Container;

final class Container implements ContainerInterface
{
    /**
     * @param array<string, string|Closure|object|callable> $definitions
     */
    public function __construct(array $definitions = []);
}
```

A definition value can be:

| Type | Meaning |
| :--- | :--- |
| Object instance | Returned as-is by `get($id)`. |
| Class-string FQCN | Autowired via `Autowire` on first `get($id)`. |
| `Closure` / callable | Invoked as a factory on first `get($id)`. |

## Public API

From `Waffle\Commons\Contracts\Container\ContainerInterface` (which extends `Psr\Container\ContainerInterface` and `ResettableInterface`):

```php
public function get(string $id): mixed;
public function has(string $id): bool;
public function set(string $id, object|callable|string $concrete): void;
public function reset(): void;
```

## Resolution algorithm — `get($id)`

1. If an instance is already cached, return it.
2. Record the id in the `resolving` stack; throw `ContainerException` if a cycle is detected.
3. Look up the definition:
   - **Closure / callable** → invoke it; cache the result.
   - **Class-string FQCN** → autowire via reflection on the constructor (see Autowire below).
   - **Object** → return directly.
4. If no definition exists → throw `NotFoundException`.
5. Cache the result, pop the resolving frame.

## Autowiring — `Waffle\Commons\Container\Autowire`

Inspects the target class's constructor parameters. For each parameter:

| Parameter kind | Resolution |
| :--- | :--- |
| Class / interface type | `Container::get($paramType)` (recursive). |
| Built-in scalar with a default value | Default value is used. |
| Nullable built-in scalar | `null`. |
| Built-in scalar without a default and not nullable | `ContainerException`. |

## Locked core services

```php
private const CORE_SERVICES = [
    \Psr\Container\ContainerInterface::class => true,
];
```

`set()` against any of these identifiers after registration throws `ContainerException` — preventing application code from swapping out the container at runtime.

## Worker-mode reset

`reset()` clears `instances` while leaving `definitions` intact. The kernel calls it between FrankenPHP worker requests so that per-request services (e.g. a request-scoped clock, user context) are rebuilt while singleton infrastructure stays in place.

## Exceptions

- `Waffle\Commons\Container\Exception\ContainerException` — implements `Psr\Container\ContainerExceptionInterface`. Thrown on resolution / autowiring failures.
- `Waffle\Commons\Container\Exception\NotFoundException` — implements `Psr\Container\NotFoundExceptionInterface`. Thrown when `get($id)` cannot find a definition.

## Quick example

```php
use Waffle\Commons\Container\Container;
use Psr\Log\LoggerInterface;

$container = new Container([
    LoggerInterface::class => new StreamLogger(),
    UserService::class      => UserService::class,           // autowired
    'app.config'           => static fn() => new Config(/*…*/),
]);

$service = $container->get(UserService::class);
$exists  = $container->has(LoggerInterface::class);          // true

$container->set('feature.flag', true);
$container->reset();                                          // clears instance cache
```
