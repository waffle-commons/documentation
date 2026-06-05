# Runtime Reference (`waffle-commons/runtime`)

> **Release:** `v0.1.0-beta2` &nbsp;|&nbsp; *No behavioural changes since Beta-1*

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

> **STAB-01 (Beta-1):** the application-side `AppKernelFactory` no longer holds a `public static GlobalsFactory $globalsFactory` — that static persisted across worker requests and made cross-request contamination possible. `WaffleRuntime` already defaults to creating its own per-process `GlobalsFactory` when none is passed, so `new WaffleRuntime()` is the canonical call from `public/index.php`.

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

## Memory-neutrality gate — Igor-PHP (`ΔM = 0`)

The worker-mode safety contract above is enforced **statically** by **Igor-PHP**, an
ultra-fast Go linter purpose-built for FrankenPHP worker mode. It is Waffle's primary
memory-neutrality gate: it parses the AST (it never executes the code) and rejects the
patterns that break the `ΔM = 0` invariant before they reach a resident worker.

**What it catches**

- **Persistent state mutation** — `static::$prop`, `$this->items[] = …` and similar
  writes that accumulate in RAM across requests.
- **Incomplete reset** — a service that implements
  `Waffle\Commons\Contracts\Service\ResettableInterface` but leaves a mutable property
  unset in `reset()`, leaking state from request *N* into request *N+1*.
- **Dangerous global access** — superglobals (`$_GET`, `$_SERVER`, …), `exit`/`die`,
  and ambient mutations such as `date_default_timezone_set()` that poison the process
  (forbidden by the statelessness mandate).

> **Symfony note:** Igor's automatic container-service audit targets Symfony projects.
> Waffle is not Symfony, so that auto-discovery does not apply; Igor still performs its
> framework-agnostic static mutation / reset / global-access analysis on our source.

**Install (in Docker), like every other gate**

```bash
docker exec -it -w /waffle-commons/runtime waffle-dev composer require --dev igor-php/igor-php
```

**Run before pushing** — mandatory for any change to memory-sensitive components
(`runtime`, `container`, `pipeline`):

```bash
docker exec -it -w /waffle-commons/runtime waffle-dev composer igor
# or, locally inside the component root:
./bin/run-igor.sh
```

**Zero baselines.** Per the Mago Purge Protocol, Igor findings are fixed, not
suppressed — do **not** commit an Igor baseline. Configuration lives in
`runtime/igor.json`; the full install and violation-resolution guide is
`runtime/igor_local_setup.md`.

## `igor:audit` execution engine

The audit is also exposed from the application console as **`igor:audit`** (the thin
`Waffle\Commons\Console\Command\MemoryAuditCommand`). Per the "every component depends
only on `waffle-commons/contracts`" rule, the command depends on the contract
`Waffle\Commons\Contracts\Runtime\AuditRunnerInterface`, while the OS-level execution
lives here in `runtime` — exactly as `db:migrate` consumes `MigrationRunnerInterface`
with the concrete in `data`.

```php
namespace Waffle\Commons\Contracts\Runtime; // contracts/src/Runtime/AuditRunnerInterface.php

interface AuditRunnerInterface
{
    /**
     * @param list<string> $arguments               flags forwarded to the script (e.g. ['--local'])
     * @param Closure(string $line, bool $isError): void $onLine
     */
    public function run(string $scriptPath, string $workingDirectory, array $arguments, Closure $onLine): int;
}
```

The concrete adapter runs the script with `proc_open` (no shell — argv array form, so
none of the lint-banned `exec`/`system`/`shell_exec`/`passthru` helpers are touched) and
streams stdout/stderr line-by-line as the audit runs:

```php
namespace Waffle\Commons\Runtime\Audit; // runtime/src/Audit/ProcessAuditRunner.php

readonly class ProcessAuditRunner implements AuditRunnerInterface
{
    public const int READ_CHUNK = 8192;          // typed class constants
    public const int SELECT_TIMEOUT_US = 200_000;
    public const int EXIT_CANNOT_EXECUTE = 127;
    // run(): proc_open + non-blocking stream_select line streaming
}
```

The argv is modelled by a hooked value object showcasing the PHP 8.5 baseline — a typed
class constant, a validating `set` property hook (throws a domain `ValidationException`),
and asymmetric visibility (`public private(set)`). A class with a writable hook cannot be
`readonly`, so it is a `final class`:

```php
namespace Waffle\Commons\Runtime\Audit; // runtime/src/Audit/IgorAuditConfig.php

final class IgorAuditConfig
{
    public const string SHELL = 'bash';

    public string $scriptPath {
        set(string $value) {
            $trimmed = trim($value);
            if ($trimmed === '') {
                throw new ValidationException('The audit script path must not be empty.', 'scriptPath');
            }
            $this->scriptPath = $trimmed;
        }
    }

    public private(set) array $arguments; // list<string>, publicly read-only

    /** @return list<string> [SHELL, scriptPath, ...arguments] */
    public function toCommand(): array;
}
```

The application wires it in `bin/waffle`:
`$app->add(new MemoryAuditCommand(new ProcessAuditRunner(), $repoRoot, $insideContainer));`.
The monorepo-wide script itself is the root `igor.sh`, also runnable as `wfl igor` — see
the devops how-to [Run checks across components](../../docs/how-to/run-checks-across-components.md#the-memory-neutrality-gate-igor-php).
