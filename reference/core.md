# Core Reference (`waffle-commons/waffle`)

The Core component provides the kernel and bootstrapping logic.

## `Waffle\Kernel`

The `Kernel` is the entry point of the application. It extends `AbstractKernel` and implements `KernelInterface`.

### `boot(): self`
Initializes the System, loads the environment (via DotEnv if not already loaded), and prepares the Container.

### `handle(ServerRequestInterface $request): ResponseInterface`
The main execution loop.
1.  Calls `boot()` and `configure()`.
2.  Checks strictly for the presence of MiddlewareStack, Container, and System.
3.  Creates a fallback `ControllerDispatcher`.
4.  Dispatches the request through the Middleware Stack.

## `Waffle\Handler\ControllerDispatcher`

This is the final handler in the chain. It uses the `Router` to match the request and executes the corresponding Controller method.
- It resolves Controller arguments using the Container (Autowiring).
- It executes the method and ensures a valid `ResponseInterface` is returned.
