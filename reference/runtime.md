# Runtime Reference (`waffle-commons/runtime`)

> **Release:** `v0.1.0-beta0`

The application runner. `WaffleRuntime` owns the request loop under FrankenPHP worker mode and falls back to a single-shot execution under the classic PHP SAPI when `frankenphp_handle_request()` is unavailable.

## `WaffleRuntime` — exact signature

```php
namespace Waffle\Commons\Runtime;

use Waffle\Commons\Contracts\Core\KernelInterface;
use Waffle\Commons\Contracts\Http\ResponseEmitterInterface;
use Waffle\Commons\Contracts\Runtime\RuntimeInterface;
use Waffle\Commons\Http\Emitter\ResponseEmitter;
use Waffle\Commons\Http\Factory\GlobalsFactory;

final class WaffleRuntime implements RuntimeInterface
{
    public function __construct(
        ?GlobalsFactory $globalsFactory = null,        // defaults to new GlobalsFactory()
        ?ResponseEmitterInterface $emitter = null,     // defaults to new ResponseEmitter()
    );

    public function loop(KernelInterface $kernel, int $maxRequests = 500): void;
}
```

The runtime contains **no concrete framework dependencies** beyond `GlobalsFactory` and `ResponseEmitter` (which themselves depend only on the http and contracts packages).

## `loop()` semantics

1. **Boot once** — `$kernel->boot()->configure()` runs exactly once when the FrankenPHP worker starts.
2. **Iterate** — up to `$maxRequests` times:
   - Under FrankenPHP: calls `frankenphp_handle_request($handler)` where `$handler`:
     - rebuilds a PSR-7 `ServerRequestInterface` from the (FrankenPHP-repopulated) superglobals via `GlobalsFactory::createFromGlobals()`,
     - calls `$kernel->handle($request)`,
     - emits the response via `ResponseEmitter::emit()`.
   - Under classic SAPI: invokes `$handler` once and exits the loop.
3. **GC** — every 50 requests, `gc_collect_cycles()` is called to keep long-running worker memory bounded.
4. **Reset on exit** — when the loop exits (max reached or FrankenPHP signaled stop), `$kernel->reset()` clears request-scoped state.

## Standard `public/index.php`

```php
<?php
declare(strict_types=1);

use Waffle\Commons\Runtime\WaffleRuntime;
use App\Factory\AppKernelFactory;

require __DIR__ . '/../vendor/autoload.php';

define('APP_ROOT', dirname(__DIR__));

$kernel = AppKernelFactory::create(
    env: getenv('APP_ENV') ?: 'prod',
    debug: getenv('APP_DEBUG') === 'true',
);

(new WaffleRuntime())->loop($kernel, maxRequests: 500);
```

## FrankenPHP / classic SAPI auto-detection

```php
$running = function_exists('frankenphp_handle_request')
    ? frankenphp_handle_request($handler)
    : false;

if (!function_exists('frankenphp_handle_request')) {
    $handler();
    break;
}
```

The runtime intentionally uses **unqualified** `function_exists` and `frankenphp_handle_request` calls so namespace-level test shims can override them via `php-mock-phpunit`. The relevant test (`tests/src/WaffleRuntimeWorkerModeTest.php`) is listed in `mago.toml [guard].excludes` for that reason.

## Worker-mode safety contract

A kernel passed to `WaffleRuntime::loop()` must:

- Be **idempotently bootable** (`boot()` and `configure()` short-circuit on subsequent calls).
- Implement `reset()` to clear request-scoped state without re-booting.
- Hold **no per-request mutable state** on the kernel object itself (use the container or request attributes for that).

The framework's `Waffle\Abstract\AbstractKernel` satisfies all three.
