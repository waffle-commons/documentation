# Quick Start Guide (`0.1.0-beta4`)

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
use Waffle\Commons\Contracts\Routing\Attribute\Route;
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

## 4. Validate input with a DTO (PHP 8.5 Property Hooks)

Waffle ships **no validation package**. Input validation is *domain logic that belongs to the value itself*, expressed with PHP 8.5 **Property Hooks**. When a controller parameter is type-hinted with a class marked `#[Dto]`, the `ControllerArgumentResolver` decodes the JSON request body, maps its keys to the constructor parameters by name, and instantiates the object — the hook does the validating.

Create `src/Dto/HelloInput.php`:

```php
<?php

declare(strict_types=1);

namespace App\Dto;

use Waffle\Commons\Contracts\Attribute\Dto;
use Waffle\Exception\ValidationException;

#[Dto]
final class HelloInput
{
    public function __construct(
        // `private(set)` makes the value read-only to callers (the DTO is
        // effectively immutable) while the `set` hook still validates on
        // hydration. A `set` hook cannot live on a `readonly` property, so
        // asymmetric visibility is the idiomatic way to get both.
        public private(set) string $name {
            set(string $value) {
                $clean = trim($value);

                if ($clean === '' || preg_match('/^\p{L}+$/u', $clean) !== 1) {
                    // ValidationException implements the
                    // ValidationExceptionInterface marker, so JsonErrorRenderer
                    // maps this to RFC 7807 HTTP 422 with `field: "name"`.
                    throw new ValidationException(
                        message: 'Field "name" must be a non-empty, alphabetic string.',
                        field: 'name',
                    );
                }

                $this->name = $clean;
            }
        },
    ) {}
}
```

Type-hint it in a controller action — hydration and validation happen *before* your code runs:

```php
#[Route(path: '/greet', name: 'greet')]
public function greet(HelloInput $input): ResponseInterface
{
    return $this->jsonResponse(data: ['message' => "Hello {$input->name}!"]);
}
```

A valid body passes straight through:

```bash
curl -k -X POST https://localhost/greet \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada"}'
# → 200 {"message":"Hello Ada!"}
```

An invalid one never reaches your action — the hook throws, and the `ErrorHandlerMiddleware` renders an RFC 7807 `422`:

```bash
curl -k -X POST https://localhost/greet \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada123"}'
```

```json
{
  "type":     "about:blank",
  "title":    "Unprocessable Entity",
  "status":   422,
  "detail":   "Field \"name\" must be a non-empty, alphabetic string.",
  "instance": "/greet"
}
```

A plain `\InvalidArgumentException` thrown from a hook is automatically unified to a `422`; throw a `Waffle\Exception\ValidationException` instead when you want the `field` key populated in the payload. See [How-To: Error Handling](../how-to/error-handling.md) for the complete mapping.

### Reusable checks with `Assert`

The hook above hand-rolls its validation — perfect when the rule is bespoke. For the common cases (e-mail, length, numeric range, IP…), `waffle-commons/utils` ships a stateless `Assert` helper that **validates *and* returns the cleansed value**, so a field collapses to a single line. It doesn't replace the property-hook philosophy; it just makes it DRY:

```php
use Waffle\Commons\Utils\Assert;

#[Dto]
final class RegistrationInput
{
    // validated, then stored trimmed + lower-cased — in one line
    public private(set) string $email {
        set => Assert::email($value);
    }

    public private(set) int $age {
        set => Assert::range($value, 18, 130);
    }

    // …
}
```

`Assert` throws a `ValidationException` that implements the same `ValidationExceptionInterface` (so it still renders as a `422`). See [How-To: Validate & Cleanse Input](../how-to/validate-input.md) and the [Utils Reference](../reference/utils.md#assert--input-validation--cleansing).

## 5. The Kernel — assembled in your `AppKernelFactory`

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

Sketch of a factory (Beta-2 wiring — CSRF subsystem inherited from Beta-1; the `OPTIONS` preflight auto-answer is enabled by passing the PSR-17 response factory to `CoreRoutingMiddleware`):

```php
use Waffle\Kernel;
use Waffle\Commons\Config\Config;
use Waffle\Commons\Container\Container;
use Waffle\Commons\Contracts\Security\Csrf\Constant as CsrfConstant;
use Waffle\Commons\Contracts\Security\Csrf\CsrfTokenManagerInterface;
use Waffle\Commons\EventDispatcher\Dispatcher\EventDispatcher;
use Waffle\Commons\EventDispatcher\Provider\ListenerProvider;
use Waffle\Commons\ErrorHandler\Middleware\ErrorHandlerMiddleware;
use Waffle\Commons\ErrorHandler\Renderer\JsonErrorRenderer;
use Waffle\Commons\Http\Factory\ResponseFactory;
use Waffle\Commons\Log\StreamLogger;
use Waffle\Commons\Pipeline\MiddlewareStack;
use Waffle\Commons\Pipeline\CoreRoutingMiddleware;
use Waffle\Commons\Pipeline\Middleware\SecureHeadersMiddleware;
use Waffle\Commons\Pipeline\Middleware\TrustedHostMiddleware;
use Waffle\Commons\Security\Csrf\CsrfTokenManager;
use Waffle\Commons\Security\Middleware\AnonymousSessionMiddleware;
use Waffle\Commons\Security\Middleware\CsrfMiddleware;
use Waffle\Commons\Security\Middleware\SecurityMiddleware;
use Waffle\Commons\Security\Security;

final class AppKernelFactory
{
    public static function create(string $env = 'prod', bool $debug = false): Kernel
    {
        $config = new Config(APP_ROOT . '/config', $env);
        $logger = new StreamLogger(streamPath: 'php://stderr');

        // SEC-01: CSRF signing secret. Config wins over env; in prod a missing
        // or short value MUST abort boot. See AppKernelFactory::resolveCsrfSecret()
        // in the skeleton for the canonical resolver.
        $csrfSecret = $config->getString('waffle.security.csrf.secret')
            ?? (getenv(CsrfConstant::SECRET_ENV_KEY) ?: null);
        $csrfTokenManager = new CsrfTokenManager(secret: $csrfSecret);

        $container = new Container([
            ConfigInterface::class => $config,
            LoggerInterface::class => $logger,
            CsrfTokenManagerInterface::class => $csrfTokenManager,
        ]);

        $responseFactory = new ResponseFactory();
        $renderer = new JsonErrorRenderer($responseFactory, debug: $debug);
        $router   = /* build via RouteDiscoverer over waffle.paths.controllers */;

        // Canonical order (unchanged since Beta-1):
        // ErrorHandler → TrustedHost → AnonymousSession → Routing → Csrf →
        // Security → SecureHeaders → Dispatcher.
        // Beta-2: CoreRoutingMiddleware receives the PSR-17 $responseFactory, so an
        // OPTIONS request to a known path is auto-answered with `204 No Content` +
        // an `Allow` header instead of dispatching a controller.
        $stack = (new MiddlewareStack())
            ->add(new ErrorHandlerMiddleware($renderer, $logger))
            ->add(new TrustedHostMiddleware($config->getArray('waffle.trusted_hosts', []) ?? []))
            ->add(new AnonymousSessionMiddleware())
            ->add(new CoreRoutingMiddleware($router, $responseFactory))
            ->add(new CsrfMiddleware($csrfTokenManager))
            ->add(new SecurityMiddleware(new Security($config), $logger))
            ->add(new SecureHeadersMiddleware());

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

`AnonymousSessionMiddleware` issues the `WAFFLE_SID` cookie that `CsrfMiddleware` binds tokens to — both pieces must be present in the stack for CSRF protection to work. See [How-To: Secure a Controller](../how-to/secure-a-controller.md) and the [CSRF explanation](../explanation/security-csrf-double-submit.md) for the design rationale.

The kernel boots lazily — the very first `handle()` call calls `boot()->configure()` if not already booted, so the factory does not need to call these explicitly.

## 6. The Runtime — `public/index.php`

The runtime is `Waffle\Commons\Runtime\WaffleRuntime`. Its public API is:

```php
public function __construct(
    GlobalsFactoryInterface $globalsFactory,   // required — wired by the app
    ResponseEmitterInterface $emitter,         // required — wired by the app
);

public function loop(KernelInterface $kernel, int $maxRequests = 500): void;
```

A complete `public/index.php`:

```php
<?php
declare(strict_types=1);

use Waffle\Commons\Http\Emitter\ResponseEmitter;
use Waffle\Commons\Http\Factory\GlobalsFactory;
use Waffle\Commons\Runtime\WaffleRuntime;
use App\Factory\AppKernelFactory;

require __DIR__ . '/../vendor/autoload.php';

define('APP_ROOT', dirname(__DIR__));

$kernel = AppKernelFactory::create(
    env: getenv('APP_ENV') ?: 'prod',
    debug: getenv('APP_DEBUG') === 'true',
);

// The app wires the concrete http factory + emitter into the agnostic runtime.
(new WaffleRuntime(new GlobalsFactory(), new ResponseEmitter()))->loop($kernel, maxRequests: 500);
```

Under FrankenPHP worker mode, `loop()`:

1. Calls `$kernel->boot()->configure()` once.
2. Repeatedly calls `frankenphp_handle_request($handler)` where `$handler` rebuilds the PSR-7 request from superglobals, calls `$kernel->handle()`, and emits the response.
3. Runs `gc_collect_cycles()` every 50 requests.
4. Calls `$kernel->reset()` when the loop exits.

Under classic PHP SAPI (when `frankenphp_handle_request` doesn't exist), the handler runs once and exits.

## 7. What's next

- Validate input natively: §4 above shows `#[Dto]` + Property Hooks; [How-To: Error Handling](../how-to/error-handling.md) covers how a hook rejection becomes an RFC 7807 `422`.
- Route a gateway catch-all: [How-To: Routing](../how-to/routing.md) documents the `priority` parameter and `{path:.*}` multi-segment matching used by the edge-gateway pattern.
- Tighten security: see [How-To: Secure a Controller](../how-to/secure-a-controller.md).
- Hook into the lifecycle: see [How-To: Events](../how-to/events.md) for `RequestReceivedEvent`, `ResponseGeneratedEvent`, `TerminateEvent`.

You now have a working Beta-2 Waffle application with full PSR-7/15/17 plumbing, RFC 7231 method handling (typed `405` + deterministic `Allow` header + `OPTIONS` preflight), Mago Zero-Debt under `composer mago`, fail-closed ABAC, stateless HMAC CSRF, and worker-mode safe runtime.
