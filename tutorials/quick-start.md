# Quick Start Guide (`v0.1.0-beta0`)

Welcome to the Waffle Framework. This guide walks you through scaffolding a new project from the `waffle-commons/skeleton` template, writing your first controller, and understanding how the Kernel + Runtime fit together.

## Prerequisites

- **PHP 8.5+** (strictly enforced — Property Hooks, Asymmetric Visibility, typed constants).
- **Composer** 2.x.
- **Docker** with Docker Compose (the skeleton ships with a FrankenPHP-based `docker-compose.yml`).
- **`ext-yaml`** (the native PECL YAML extension — the `waffle-dev` Docker image ships with it).

## 1. Create the project

```bash
composer create-project waffle-commons/skeleton my-app
cd my-app
docker compose up -d
```

The default skeleton serves on `https://localhost/` (FrankenPHP terminates TLS with a self-signed cert).

## 2. Write your first controller

Create `src/Controller/HelloController.php`:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Waffle\Commons\Routing\Attribute\Route;
use Waffle\Core\BaseController;

final class HelloController extends BaseController
{
    #[Route(path: '/hello/{name}', name: 'hello')]
    public function index(string $name): ResponseInterface
    {
        return $this->jsonResponse(data: ['message' => "Hello $name"]);
    }
}
```

`BaseController::jsonResponse()` returns a PSR-7 `ResponseInterface` with `Content-Type: application/json`.

## 3. Hit the endpoint

```bash
curl -k https://localhost/hello/Waffle
```

Expected output:

```json
{"message":"Hello Waffle"}
```

## 4. The Kernel — assembled in your `AppKernelFactory`

The skeleton's `src/Factory/AppKernelFactory.php` wires every component the kernel needs. The setter contract is verbatim from `Waffle\Abstract\AbstractKernel`:

```php
public function setContainerImplementation(PsrContainerInterface $container): void;
public function setConfiguration(ConfigInterface $config): void;
public function setSecurity(SecurityInterface $security): void;
public function setMiddlewareStack(MiddlewareStackInterface $stack): void;
public function setEventDispatcher(EventDispatcherInterface $dispatcher): void;
```

The PSR-3 logger is passed to the constructor (default `NullLogger`):

```php
public function __construct(protected LoggerInterface $logger = new NullLogger())
```

Sketch of a factory:

```php
use Waffle\Kernel;
use Waffle\Commons\Config\Config;
use Waffle\Commons\Container\Container;
use Waffle\Commons\EventDispatcher\Dispatcher\EventDispatcher;
use Waffle\Commons\EventDispatcher\Provider\ListenerProvider;
use Waffle\Commons\ErrorHandler\Middleware\ErrorHandlerMiddleware;
use Waffle\Commons\ErrorHandler\Renderer\JsonErrorRenderer;
use Waffle\Commons\Http\Factory\ResponseFactory;
use Waffle\Commons\Log\StreamLogger;
use Waffle\Commons\Pipeline\MiddlewareStack;
use Waffle\Commons\Pipeline\CoreRoutingMiddleware;
use Waffle\Commons\Pipeline\Middleware\TrustedHostMiddleware;
use Waffle\Commons\Security\Security;

final class AppKernelFactory
{
    public static function create(string $env = 'prod', bool $debug = false): Kernel
    {
        $config = new Config(APP_ROOT . '/config', $env);
        $logger = new StreamLogger(streamPath: 'php://stderr');

        $container = new Container([
            ConfigInterface::class => $config,
            LoggerInterface::class => $logger,
        ]);

        $renderer = new JsonErrorRenderer(new ResponseFactory(), debug: $debug);
        $router   = /* build via RouteDiscoverer over waffle.paths.controllers */;
        $stack    = (new MiddlewareStack())
            ->add(new ErrorHandlerMiddleware($renderer, $logger))
            ->add(new TrustedHostMiddleware($config->getArray('waffle.trusted_hosts', []) ?? []))
            ->add(new CoreRoutingMiddleware($router));

        $kernel = new Kernel($logger);
        $kernel->setConfiguration($config);
        $kernel->setContainerImplementation($container);
        $kernel->setSecurity(new Security($config));
        $kernel->setMiddlewareStack($stack);
        $kernel->setEventDispatcher(new EventDispatcher(new ListenerProvider()));

        return $kernel;
    }
}
```

The kernel boots lazily — the very first `handle()` call calls `boot()->configure()` if not already booted, so the factory does not need to call these explicitly.

## 5. The Runtime — `public/index.php`

The runtime is `Waffle\Commons\Runtime\WaffleRuntime`. Its public API is:

```php
public function __construct(
    ?GlobalsFactory $globalsFactory = null,
    ?ResponseEmitterInterface $emitter = null,
);

public function loop(KernelInterface $kernel, int $maxRequests = 500): void;
```

A complete `public/index.php`:

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

Under FrankenPHP worker mode, `loop()`:

1. Calls `$kernel->boot()->configure()` once.
2. Repeatedly calls `frankenphp_handle_request($handler)` where `$handler` rebuilds the PSR-7 request from superglobals, calls `$kernel->handle()`, and emits the response.
3. Runs `gc_collect_cycles()` every 50 requests.
4. Calls `$kernel->reset()` when the loop exits.

Under classic PHP SAPI (when `frankenphp_handle_request` doesn't exist), the handler runs once and exits.

## 6. What's next

- Add real validation: see [How-To: Routing](../how-to/routing.md) for `#[Argument]` parameter injection.
- Tighten security: see [How-To: Secure a Controller](../how-to/secure-a-controller.md).
- Hook into the lifecycle: see [How-To: Events](../how-to/events.md) for `RequestReceivedEvent`, `ResponseGeneratedEvent`, `TerminateEvent`.

You now have a working Beta 0 Waffle application with full PSR-7/15/17 plumbing, Mago Zero-Debt under `composer mago`, and worker-mode safe runtime.
