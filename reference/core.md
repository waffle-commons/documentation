# Core Reference (`waffle-commons/waffle`)

The Core component acts as the nervous system of the Waffle framework. It bootstraps the application, handles the request/response lifecycle, and integrates the primary components (Container, Config, Security, Pipeline, and Event Dispatcher).

## The Kernel

The `Waffle\Kernel` (extending `Waffle\Abstract\AbstractKernel`) is the entry point of the application. It uses PHP 8.5 **Asymmetric Visibility** for internal state management:

```php
abstract class AbstractKernel implements KernelInterface
{
    protected(set) ?System $system = null;
    protected(set) ?MiddlewareStackInterface $middlewareStack = null;
    // ...
}
```

### Lifecycle & Events

The Kernel follows a strict lifecycle, dispatching PSR-14 events at key stages:

1.  **Boot**: Initializes environmental variables.
2.  **Configure**: Loads configuration, sets up the Container, and initializes the System.
3.  **Handle**: Processes the incoming `ServerRequestInterface`.
    - **Event**: `Waffle\Event\RequestReceivedEvent` is dispatched before the pipeline.
    - **Processing**: Request passes through the `MiddlewareStack`.
    - **Event**: `Waffle\Event\ResponseGeneratedEvent` is dispatched after the pipeline.
4.  **Terminate**: Dispatched after the response is emitted to the client.
    - **Event**: `Waffle\Event\TerminateEvent` for heavy asynchronous tasks.

### Key Methods

#### `handle(ServerRequestInterface $request): ResponseInterface`

This is the main entry point for handling HTTP requests. It ensures the kernel is booted and configured before passing the request to the Middleware Stack. It dispatches lifecycle events to allow hooks into the request processing.

```php
public function handle(ServerRequestInterface $request): ResponseInterface
{
    // ...
    $this->dispatch(new RequestReceivedEvent($request));
    // ... pipeline execution ...
    $this->dispatch(new ResponseGeneratedEvent($response));
    return $response;
}
```

#### `boot(): static`

Initializes the environment and sets up the base state.

#### `configure(): void`

Sets up the application state, initializes the Container with services and controllers, and boots the `System`.

## Dependency Injection

The Kernel relies on Setter Injection for its core dependencies, allowing for flexibility:

```php
$kernel->setConfiguration($config);
$kernel->setSecurity($security);
$kernel->setContainerImplementation($secureContainer);
$kernel->setMiddlewareStack($stack);
$kernel->setEventDispatcher($dispatcher); // New in Alpha 5
```
