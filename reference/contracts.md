# Contracts Reference (`waffle-commons/contracts`)

This package contains the interfaces that define the Waffle Architecture. It ensures decoupling between components.

## Key Interfaces

### `Waffle\Commons\Contracts\Core\KernelInterface`
Defines the `handle()`, `boot()`, and `configure()` methods.

### `Waffle\Commons\Contracts\Container\ContainerInterface`
Extends `Psr\Container\ContainerInterface` and adds a `set()` method for mutable containers (compatible with Waffle's Container).

### `Waffle\Commons\Contracts\Security\SecurityInterface`
Defines the `analyze(object $object)` method used by the Secure Container.

### `Waffle\Commons\Contracts\Pipeline\MiddlewareStackInterface`
Defines methods to manage the middleware stack:
- `add(MiddlewareInterface $middleware)`
- `prepend(MiddlewareInterface $middleware)`
- `createHandler(RequestHandlerInterface $fallback)`
