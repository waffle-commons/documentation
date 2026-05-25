# Routing Reference (`waffle-commons/routing`)

> **Release:** `v0.1.0-beta1`

Attribute-driven router. Routes live next to the controller code via the `#[Route]` attribute; the `RouteDiscoverer` scans the configured controller directory at boot time and the compiled table is cached.

## `#[Route]` attribute — exact signature

From `src/Attribute/Route.php`:

```php
namespace Waffle\Commons\Routing\Attribute;

use Attribute;

#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
final class Route
{
    /**
     * @param array<Argument>|null $arguments
     * @param int                  $priority Higher matches first. Use negative values
     *                                       (e.g. `-1000`) for catch-all routes that
     *                                       must be evaluated last. Defaults to `0`.
     */
    public function __construct(
        public string $path,
        public ?string $name = null,
        public ?array $arguments = null,
        public int $priority = 0,
    ) {}
}
```

| Parameter | Type | Default | Purpose |
| :--- | :--- | :--- | :--- |
| `path` | `string` | (required) | URL pattern. Use `{name}` for placeholders, `{name:regex}` for a constrained placeholder, and `{name:.*}` for a multi-segment catch-all — captured values land in `$request->getAttribute('_route_params')`. |
| `name` | `?string` | `null` | Unique route name. Used for debugging and the `route:list` console command. |
| `arguments` | `?array<Argument>` | `null` | Container-resolved argument injection (see below). |
| `priority` | `int` | `0` | Match order. Routes are sorted by **descending** priority at boot; higher matches first, negative values (e.g. `-1000`) run last as catch-all fallbacks. |

Targets both `TARGET_CLASS` and `TARGET_METHOD`. Class-level `#[Route]` defines a path prefix for every method-level `#[Route]` on that class.

## `#[Argument]` attribute — exact signature

From `src/Attribute/Argument.php`:

```php
namespace Waffle\Commons\Routing\Attribute;

use Attribute;

#[Attribute]
final class Argument
{
    public function __construct(
        public string $classType,
        public string $paramName,
        public bool $required = true,
    ) {}
}
```

Tells the router that the controller's `$paramName` argument should be resolved from the container by FQCN `$classType`. Use this when you want a controller method to receive a service that isn't naturally derivable from the route path:

```php
use Waffle\Commons\Routing\Attribute\Route;
use Waffle\Commons\Routing\Attribute\Argument;

#[Route(
    path: '/users',
    name: 'users.create',
    arguments: [
        new Argument(classType: UserRepository::class, paramName: 'repo'),
    ],
)]
public function create(ServerRequestInterface $request, UserRepository $repo): ResponseInterface
{
    // $repo is fetched from the container at dispatch time
}
```

## `RouterInterface`

From `Waffle\Commons\Contracts\Routing\RouterInterface`:

```php
public function boot(ContainerInterface $container): static;

public function matchRequest(ServerRequestInterface $request): ?MatchedRoute;

/** @return list<MatchedRoute> */
public function getRoutes(): array;
```

`matchRequest()` returns `null` when no route matches — `CoreRoutingMiddleware` converts that to the concrete `Waffle\Commons\Contracts\Routing\Exception\RouteNotFoundException` (rendered as RFC 7807 `404` by the error handler).

### `MatchedRoute` DTO

The router exposes its result as the immutable `Waffle\Commons\Contracts\Routing\MatchedRoute` DTO (Beta-1 leftover-purge §1): every consumer reads typed properties instead of decomposing a nested associative array.

```php
final readonly class MatchedRoute
{
    public function __construct(
        public string $className,   // class-string of the controller
        public string $method,      // method name on the controller
        public array  $arguments,   // per-argument metadata from #[Argument]
        public string $path,        // original route path (e.g. "/users/{id}")
        public string $name,        // route name from #[Route(name: ...)]
        public array  $params = [], // path parameters extracted for this request
        public int    $priority = 0, // higher matches first; -1000 = catch-all
    ) {}

    public function withParams(array $params): self;
}
```

Read it via property access: `$match->className`, `$match->params['id']`, etc. The previous nested-array return shape has been removed — type-system enforcement now catches every typo at compile time instead of runtime.

## Path placeholders & PCRE compilation

`Router::compilePattern()` (memoised once per worker) compiles each route path into a PCRE with named capture groups. Three placeholder forms are supported:

| Placeholder | Compiles to | Matches |
| :--- | :--- | :--- |
| `{id}` | `(?P<id>[^/]+)` | a single URL segment (the default constraint) |
| `{id:\d+}` | `(?P<id>\d+)` | one segment restricted by a custom regex constraint |
| `{path:.*}` | `(?P<path>.*)` | **multiple** segments — spans `/`, the basis for catch-all routing |

Static text between placeholders is `preg_quote`d, so literal dots and dashes in a path stay literal (`/v1.0/status` only matches `/v1.0/status`, never `/v1x0/status`). A custom constraint containing a literal `}` is unsupported.

## Priority & catch-all routes

Beta-1 Phase 3 introduces the `priority` field on `#[Route]`. Higher values match first; the router sorts the compiled table by descending priority at boot time and caches the sorted collection so subsequent boots skip the sort. The default priority is `0`, so unannotated routes keep their declaration order within a single priority bucket.

```php
#[Route(path: '/', name: 'app')]
final class AppController
{
    #[Route(path: 'users/{id:\d+}', name: 'user_show')]   // constrained: digits only
    public function show(int $id): array { /* ... */ }
}

// EcoShield Gateway forward: a multi-segment catch-all that ONLY matches when no
// higher-priority route claimed the URI. `{forwarded:.*}` spans every remaining
// segment, and the negative priority guarantees it sits at the tail of the table.
#[Route(path: '/', name: 'gateway', priority: -1000)]
final class GatewayController
{
    #[Route(path: '{forwarded:.*}', name: 'fallback')]
    public function forward(string $forwarded): ResponseInterface { /* proxy to legacy monolith */ }
}
```

**Method-level wins** — if a method's `#[Route]` declares a non-zero `priority`, it overrides the class-level value. If only the class declares one, every method inherits it. This mirrors how the `path` is already composed (class prefix + method suffix).

**Multi-segment catch-all** — the `{name:.*}` form matches across `/` boundaries, so a single gateway route can absorb arbitrarily deep URIs. Combine it with a negative `priority` so it only fires after every specific route has been tried.

## Components

| Class | Responsibility |
| :--- | :--- |
| `Waffle\Commons\Routing\Router` | The router entry point. Boots from the container, compiles the route table. |
| `Waffle\Commons\Routing\RouteDiscoverer` | Walks the controller directory (`waffle.paths.controllers` in `app.yaml`), resolves each file's FQCN via `Waffle\Commons\Utils\Service\ClassParser`, and reads `#[Route]` attributes via native `ReflectionClass`/`ReflectionMethod`. |
| `Waffle\Commons\Routing\ControllerFinder` | Filesystem traversal of `*.php` files. |
| `Waffle\Commons\Routing\RouteParser` | Converts a `Route` attribute + native reflection metadata into a `MatchedRoute` DTO (method-level `priority` overrides the class-level default). |
| `Waffle\Commons\Routing\Trait\RequestTrait` | Helpers for extracting routing metadata (`_controller`, `_route_params`) from a PSR-7 request. |

## Configuration

```yaml
# config/app.yaml
waffle:
  paths:
    controllers: src/Controller
```

`RouteDiscoverer` is rooted at `APP_ROOT . '/' . waffle.paths.controllers` and recursively scans for `*.php` files containing controllers with `#[Route]` attributes.

## Request attributes after routing

When `CoreRoutingMiddleware` runs successfully, the following attributes are present on the request when the controller is invoked:

- `_controller` — the FQCN of the resolved controller class.
- `_method` — the controller method name to invoke.
- `_route_params` — `array<string, mixed>` of `{placeholder}` values extracted from the path.

The `Waffle\Commons\Routing\Trait\RequestTrait` provides typed accessors for these.
