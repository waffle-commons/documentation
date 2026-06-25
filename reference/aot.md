# AOT Compilation Reference (Beta-5)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *Ahead-of-Time container + route compilation (AOT-01 / AOT-02, RFC-019)*
> **Spans:** `waffle-commons/contracts`, `waffle-commons/console`, `waffle-commons/routing`, `waffle-commons/waffle`.

Build-time Ahead-of-Time compilation for FrankenPHP's resident worker. Two independent artifacts — a reflection-free **compiled container** and a serialised **route trie** — are produced by explicit CLI commands and consumed only when `WAFFLE_AOT=1`, with a transparent reflection fallback on any miss. The compiled container is proven graph-identical to the runtime container by a snapshot test.

For the architectural reasoning (why a build step, why no fingerprinting), see [Explanation: Ahead-of-Time Compilation](../explanation/aot-compilation.md). To switch it on, see [How to: Enable AOT](../how-to/enable-aot.md).

## Contract — `Waffle\Commons\Contracts\Container\CompiledContainerInterface`

A marker interface that extends `Waffle\Commons\Contracts\Container\ContainerInterface` (and therefore PSR-11 `Psr\Container\ContainerInterface` + `Waffle\Commons\Contracts\Service\ResettableInterface`):

```php
namespace Waffle\Commons\Contracts\Container;

interface CompiledContainerInterface extends ContainerInterface {}
```

It adds no methods — a compiled container is a drop-in `ContainerInterface`. The marker lets the kernel's loader recognise an artifact as a valid compiled container before swapping it in. The inherited surface (`get()`, `has()`, `set()`, `reset()`) is implemented by the generated class.

## Container compiler — `Waffle\Commons\Console\Compiler\ContainerCompiler`

`final class`. Given a fully-booted, locked runtime container, emits PHP source for a generated class implementing `CompiledContainerInterface`.

```php
public const string DEFAULT_RUNTIME_CONTAINER = 'Waffle\\Commons\\Container\\Container';

public function compile(
    ContainerInterface $container,                  // a booted, locked runtime container
    string $namespace = 'Waffle\\Generated',
    string $className = 'CompiledContainer',
    string $runtimeContainer = self::DEFAULT_RUNTIME_CONTAINER,
): string;                                          // generated source, including the <?php tag
```

- Reads the container's private `definitions` map via `ReflectionObject` — so the compiler depends only on the container **contract**, never the concrete `Waffle\Commons\Container` package. A container that exposes no inspectable `definitions` array throws a `Waffle\Commons\Console\Exception\CompilerException`.
- **Inlinable** definitions (class-string concretes for an *instantiable* class whose every constructor parameter is statically resolvable) are wired with a literal `new \FQCN(...)` expression. **Closures** (lazy factories), **pre-built objects**, and non-instantiable/abstract classes are *passthroughs* — the generated `get()` delegates them to the composed runtime container.
- Argument emission mirrors `Waffle\Commons\Container\Autowire::resolveDependencies()` exactly: a nullable, unregistered dependency emits `$this->has($id) ? $this->get($id) : null`; a non-nullable one emits a bare `$this->get($id)` (so it throws identically for an unknown id); a union-typed parameter emits a `has()`-guarded ternary chain that picks the first *registered* member left-to-right, falling back to the declared default. Variadics are skipped; primitive parameters use their default literal.

> Source is assembled as hand-rolled strings (no `nikic/php-parser` / codegen dependency), keeping `console` inside its contracts-only dependency perimeter.

### Generated class shape

The emitted class (default FQCN `Waffle\Generated\CompiledContainer`) composes the concrete runtime container and is reset-safe:

```php
final class CompiledContainer implements \Waffle\Commons\Contracts\Container\CompiledContainerInterface
{
    private array $instances = [];                  // per-request memo, cleared on reset()

    public function __construct(private readonly \Waffle\Commons\Container\Container $runtime) {}

    public function get(string $id): mixed;          // match($id) of inlined `new …`; default → $this->runtime->get($id)
    public function has(string $id): bool;           // → $this->runtime->has($id)
    public function set(string $id, object|callable|string $concrete): void; // → $this->runtime->set(...)
    public function reset(): void;                   // resets memoised ResettableInterface services, then $this->runtime->reset()
}
```

`reset()` cascades `reset()` to every memoised service implementing `Waffle\Commons\Contracts\Service\ResettableInterface`, clears `$instances`, then resets the composed runtime container — introducing no new cross-request state (`igor-php` clean).

## Container compile command — `Waffle\Commons\Console\Command\ContainerCompileCommand`

`final readonly`. Console name **`container:compile`**. Synopsis: `container:compile [<artifact-path>]`.

```php
public function __construct(
    ContainerInterface $container,                  // booted, locked runtime container
    ContainerCompiler $compiler = new ContainerCompiler(),
    string $namespace = 'Waffle\\Generated',
    string $className = 'CompiledContainer',
    string $runtimeContainer = ContainerCompiler::DEFAULT_RUNTIME_CONTAINER,
);
```

Runs the compiler over the wired container and writes the generated class to disk. Default artifact path **`var/cache/CompiledContainer.php`**, overridable with the positional `artifact-path` argument; the target directory is created (`0775`) if absent. Returns `ExitCode::SUCCESS` (`0`) on success, `ExitCode::CONFIG` (`78`) when the directory or file cannot be written, and `ExitCode::FAILURE` (`1`) when compilation throws (the error goes to stderr).

## Kernel fast-path loader — `Waffle\Factory\CompiledContainerLoader`

`final readonly`. The AOT-01 hook inside `Waffle\Abstract\AbstractKernel::configure()`: after the runtime container is built and **locked**, the kernel calls `->load()` to (conditionally) swap in the compiled container.

```php
public const string ENV_FLAG = 'WAFFLE_AOT';                         // opt-in flag
public const string DEFAULT_ARTIFACT = 'var/cache/CompiledContainer.php';
public const string DEFAULT_CLASS = 'Waffle\\Generated\\CompiledContainer';
public const string REBUILD_COMMAND = 'bin/waffle container:compile';

public function __construct(
    string $artifactPath,
    string $compiledClass = self::DEFAULT_CLASS,
    LoggerInterface $logger = new NullLogger(),
);

public function load(ContainerInterface $runtime): ContainerInterface;
```

`load()` returns the compiled container wrapping `$runtime` **only when all hold**:

1. `WAFFLE_AOT` is set to an on-value — one of `1`, `true`, `on`, `yes` (case-insensitive). Any other value, or an unset variable, keeps the reflection path.
2. The artifact file exists at `$artifactPath`.
3. `require_once`-ing it defines `$compiledClass`, the class implements `CompiledContainerInterface`, and `new $compiledClass($runtime)` constructs without throwing.

On **any** miss — disabled, missing/corrupt artifact, wrong class, construction failure — it logs a `warning()` and returns `$runtime` **unchanged** (RFC-019 mandatory fallback). On a **successful** load it logs a prominent `warning()` that the artifact is *not* validated against the current code and **must** be regenerated with `bin/waffle container:compile` after any code change (AOT-04 — no fingerprinting by design). The loader depends only on the container contract; the generated class composes the concrete `Container` internally.

## Route trie — `Waffle\Commons\Routing\Trie\RouteTrie`

`final class`. A segment-keyed lookup tree resolving routes in **O(depth)** instead of the sequential `foreach` + per-route PCRE of `Waffle\Commons\Routing\Router::resolve()` (AOT-02).

```php
namespace Waffle\Commons\Routing\Trie;

public static function build(array $routes): self;          // list<MatchedRoute>, priority-sorted
public static function fromArray(array $data): self;        // rehydrate from toArray()
public function toArray(): array;                           // flatten to a plain nested array
public function match(string $method, string $path): ?MatchedRoute; // null on path miss; throws MethodNotAllowedException on method-only mismatch
```

- Routes index by path segment: a literal segment is a `static` key; a `{name}` / `{name:constraint}` segment is the single dynamic child; a `{name:.*}`-style segment (a constraint that can span `/`) is a catch-all leaf consuming the path remainder.
- **Priority parity:** `build()` tags each route with its index in the priority-sorted list; `match()` collects all path-matching candidates from every branch and sorts by that build-time order, so the trie's winner is identical to the sequential matcher's. 405 `Allow`-set construction and HEAD/OPTIONS handling are ported verbatim from the `Router`.
- **Root catch-all (AOT-03):** a root-mounted catch-all (`/{path:.*}`) is evaluated even when the path is exhausted, so it matches the root path `/`.
- `toArray()` keeps `MatchedRoute` instances in place (each paired with its build-time `order`), so a `serialize()` + base64 dump round-trips exactly without a `__set_state()` hook. Nodes are mutable only during `build()` and frozen for the worker lifetime thereafter.

Each node is a `Waffle\Commons\Routing\Trie\TrieNode` (`final`, `@internal`); `TrieNode::fromArray()` defensively narrows every thawed field so a stale artifact cannot smuggle an unexpected type into the matcher.

### Router integration

`Waffle\Commons\Routing\Router` holds a nullable `RouteTrie` (prebuilt at `boot()` or loaded from cache) and, in `resolve()`, prefers it whenever present:

- Cache key `waffle.routes.trie` (`TRIE_CACHE_KEY`) — a prebuilt trie array rehydrated via `RouteTrie::fromArray()`.
- Cache key `waffle.routes.discovered` (`CACHE_KEY`) — the priority-sorted `MatchedRoute` list; when only this is cached, the router rebuilds the trie from it (`RouteTrie::build()`).
- With no trie at all (routes seeded directly without a boot pass), `resolve()` falls back to the sequential matcher — observably equivalent.

## Route compile command — `Waffle\Commons\Console\Command\RouteCompileCommand`

`final readonly`. Console name **`route:compile`**. Synopsis: `route:compile [<artifact-path>]`.

```php
public function __construct(
    RouterInterface $router,
    ?Closure $trieCompiler = null,                  // Closure(list<MatchedRoute>): array<string, mixed>
);
```

Takes the priority-sorted route list from `RouterInterface::getRoutes()` and writes a PHP artifact that `return`s the rehydrated payload (`base64_decode` + `unserialize`). When the optional `$trieCompiler` closure is wired (the app may reference `RouteTrie::build($routes)->toArray()`), the serialised payload is the trie array; otherwise the route list itself is serialised and the router rebuilds the trie at boot — identical behaviour either way (transparency + mandatory fallback).

Default artifact path **`var/cache/routes.trie.php`**, overridable with the positional `artifact-path` argument. Returns `ExitCode::SUCCESS` (`0`), `ExitCode::CONFIG` (`78`) on a write failure, or `ExitCode::FAILURE` (`1`) when discovery throws.

> Perimeter note: `RouteCompileCommand` depends only on the routing **contracts** (`RouterInterface`, `MatchedRoute`), never on the concrete `RouteTrie` — producing the trie array is delegated to the app-wired closure.

## Exceptions

- `Waffle\Commons\Console\Exception\CompilerException` — raised by `ContainerCompiler::compile()` when the container exposes no inspectable `definitions` map (or the property is not an array).

## Worker-safety contract

The compiled container introduces no cross-request state beyond its per-request memo (reset every request, like the runtime container's instance cache); the route trie is immutable once built. Both AOT paths are off by default — only `WAFFLE_AOT=1` consults the compiled container, and the trie is consulted only when present in cache or built at boot. The `igor-php` worker-mode audit passes with zero findings.

## Quick example

```bash
# Build the artifacts as a deploy step (CLI only — never during a request)
php bin/waffle container:compile               # → var/cache/CompiledContainer.php
php bin/waffle route:compile                   # → var/cache/routes.trie.php

# Enable the fast path for the resident worker
WAFFLE_AOT=1 frankenphp run
```

With `WAFFLE_AOT` unset, the kernel boots on the reflection container and live route discovery — the default dev path, entirely unaffected.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
