# Core Reference (`waffle-commons/waffle`)

The Core component acts as the nervous system of the Waffle framework. It bootstraps the application, handles the request/response lifecycle, and integrates the primary components (Container, Config, Security, Pipeline).

## The Kernel

The `Waffle\Kernel` (extending `Waffle\Abstract\AbstractKernel`) is the entry point of the application.

### Lifecycle

The Kernel follows a strict lifecycle:

1.  **Boot**: Initializes environmental variables.
2.  **Configure**: Loads configuration, sets up the Container, and initializes the System.
3.  **Handle**: Processes the incoming `ServerRequestInterface` and returns a `ResponseInterface`.

### Key Methods

#### `handle(ServerRequestInterface $request): ResponseInterface`

This is the main entry point for handling HTTP requests. It ensures the kernel is booted and configured before passing the request to the Middleware Stack.

```php
use Waffle\Commons\Runtime\WaffleRuntime;

// The Runtime calls handle() internally
$runtime->run($kernel, $request, $emitter);
```

#### `boot(): self`

Initializes the environment (e.g., loading `.env` variables if not present).

#### `configure(): self`

Sets up the application state. It:
- Validates that Config and Security are injected.
- Initializes the Service and Controller factories within the Container.
- Boots the core System.

## Dependency Injection

The Kernel relies on Setter Injection for its core dependencies, allowing for flexibility in how the application is assembled (e.g., by a Factory).

```php
// From App\Factory\AppKernelFactory
$kernel->setConfiguration($config);
$kernel->setSecurity($security);
$kernel->setContainerImplementation($secureContainer);
$kernel->setMiddlewareStack($stack);
```
