# The Request Lifecycle

Understanding how Waffle handles a request is key to mastering the framework.

## 1. Runtime
The `WaffleRuntime` receives the request.
- If using **FrankenPHP**, the application stays resides memory.
- It instantiates and boots the `Kernel`.

## 2. Kernel Boot
The `Kernel` initializes the core services:
- **Config**: Loads `app.yaml` and environment variables.
- **Container**: Builds the dependency injection container.
- **System**: Enforces the SecureContainer decoration.

## 3. Pipeline
The `MiddlewareStack` processes the request in FIFO order.
- Global Middleware (e.g., Error Handling) runs here.

## 4. Router
The `ControllerDispatcher` (the final handler) calls the `Router`.
- Matches the URL against `#[Route]` attributes.
- Resolves the Controller class and method.

## 5. Security
Before the Controller is instantiated, the **SecureContainer** validates the object against the configured Security Rules (Levels 1-10).

## 6. Controller
The Controller method executes and returns a `ResponseInterface`, which flows back through the pipeline to the client.
