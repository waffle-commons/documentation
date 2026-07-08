# How to Enable AOT Compilation

Waffle's Ahead-of-Time compilation (AOT-01 / AOT-02, RFC-019) moves container wiring and route discovery out of the request path and into an explicit build step, for sub-millisecond cold starts under FrankenPHP's resident worker. It is **opt-in and transparent**: two CLI commands produce artifacts, one environment flag (`WAFFLE_AOT=1`) tells the kernel to use them, and any miss falls back to the reflection path with a logged warning. With the flag unset — the default and the whole dev experience — nothing changes.

This recipe wires the two compile commands, builds the artifacts, and switches the fast path on. For the API surface see the [AOT reference](../reference/aot.md); for the reasoning see [Explanation: Ahead-of-Time Compilation](../explanation/aot-compilation.md).

## 1. Wire the compile commands

The two commands are registered explicitly in your console binary (no auto-discovery). `container:compile` needs the **booted, locked** runtime container; `route:compile` needs the booted router:

```php
use Waffle\Commons\Console\Command\ContainerCompileCommand;
use Waffle\Commons\Console\Command\RouteCompileCommand;
use Waffle\Commons\Routing\Trie\RouteTrie;

// $container is the application's runtime container, booted and locked.
$app->add(new ContainerCompileCommand(container: $container));

// $router has run discovery (RouteDiscoverer/RouteParser) at boot.
$app->add(new RouteCompileCommand(
    router: $router,
    // Optional: build the trie array directly. Omit to serialise the route
    // list — the router rebuilds the trie at boot either way.
    trieCompiler: static fn(array $routes): array => RouteTrie::build($routes)->toArray(),
));
```

> The `trieCompiler` closure is optional. Without it, `route:compile` serialises the priority-sorted route list and the `Router` rebuilds the trie at boot — behaviour is identical. The closure only lets you cache the *already-flattened* trie so boot skips even the rebuild. `RouteCompileCommand` depends only on the routing contracts, so the concrete `RouteTrie` reference stays in your application wiring.

## 2. Build the artifacts

Run the two commands from the project root so the relative artifact paths resolve. **This is a CLI-only build step — never run it during an HTTP request.**

```bash
# inside the container
docker exec -w /waffle-commons/your-app waffle-dev php bin/waffle container:compile
docker exec -w /waffle-commons/your-app waffle-dev php bin/waffle route:compile

# or, locally
php bin/waffle container:compile
php bin/waffle route:compile
```

Each prints a one-line summary:

```
Compiling container to "var/cache/CompiledContainer.php"… done (Waffle\Generated\CompiledContainer).
Compiling routes to "var/cache/routes.trie.php"… done (12 route(s)).
```

This writes:

- `var/cache/CompiledContainer.php` — the generated `Waffle\Generated\CompiledContainer` class with hardcoded constructor wiring.
- `var/cache/routes.trie.php` — the serialised route trie (`base64_decode` + `unserialize` on load).

Override the default path with a positional argument, e.g. `php bin/waffle container:compile build/CompiledContainer.php`.

Both commands return `ExitCode::SUCCESS` (`0`) on success, `ExitCode::CONFIG` (`78`) if the target cannot be written, and `ExitCode::FAILURE` (`1`) on a compilation error (written to stderr), so a CI/CD pipeline can branch on the result.

## 3. Switch the fast path on

The kernel consults the compiled container only when `WAFFLE_AOT` is set to an on-value (`1`, `true`, `on`, or `yes`, case-insensitive). Set it where your worker process is launched:

```dotenv
# .env (local) / orchestrator env (production)
WAFFLE_AOT=1
```

```bash
WAFFLE_AOT=1 frankenphp run
```

On the next boot, `Waffle\Factory\CompiledContainerLoader` (invoked from `AbstractKernel::configure()` right after the container is locked) loads `var/cache/CompiledContainer.php`, confirms it implements `CompiledContainerInterface`, and swaps it in for the reflection container. The route trie is loaded from cache independently and used by the `Router` whenever present — it does not require the env flag.

## How it works

- **Container.** The `ContainerCompiler` reads the locked container's service definitions and emits a class that resolves *inlinable* services (class-string concretes with statically-resolvable constructors) through literal `new …` calls, delegating closures and pre-built objects to the composed runtime container. The compiled graph is proven identical to the runtime graph by a snapshot test.
- **Routes.** The `RouteTrie` resolves matches in O(depth) by indexing routes on their path segments, reproducing the sequential matcher's priority ordering, 405 handling, and HEAD/OPTIONS semantics exactly.
- **Fallback.** A disabled flag, a missing or corrupt artifact, the wrong class, or a construction failure all degrade to the reflection path — the loader logs a `warning()` and returns the runtime container unchanged.

## ⚠️ Regenerate after every code change

The loader does **not** fingerprint the artifact against your code (AOT-04, by design — no brittle invalidation daemon). A stale artifact silently serves an **outdated service graph**. So:

- Every successful AOT load emits a prominent `LoggerInterface::warning()` reminding you that the artifact is not validated against the current code and **must** be regenerated. Treat that warning as a checklist item, not noise.
- Make `container:compile` and `route:compile` a **build/deploy step** that runs after every code change — alongside `composer install --no-dev` and OPcache warm-up — never a one-off you forget. If in doubt, delete the artifacts: a missing artifact falls back cleanly to reflection.

```bash
# Recommended deploy ordering
composer install --no-dev --optimize-autoloader
php bin/waffle container:compile
php bin/waffle route:compile
# … then start the worker with WAFFLE_AOT=1
```

## Turning it off

Unset `WAFFLE_AOT` (or set it to anything other than an on-value) and the kernel boots on the reflection container with live route discovery — exactly as it did before AOT. The artifacts on disk are inert when the flag is off, so leaving them in place is harmless.

See the [AOT reference](../reference/aot.md) for the compiler and loader APIs, and the [console reference](../reference/console.md) for the full command surface.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
