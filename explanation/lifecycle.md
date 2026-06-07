# The Request Lifecycle

Understanding how Waffle handles a request is key to mastering the framework.

## 1. Runtime
The `WaffleRuntime` receives the request.
- If using **FrankenPHP**, the application stays resident in memory across requests.
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

## 7. Reset (worker mode)
Under FrankenPHP the process does **not** die after the response — it loops back to step 1 for the next request. Any state left mutated on a shared service would survive into that next request, so the runtime releases request-scoped state every cycle: services implementing `ResettableInterface` are reset and PSR-7 resources are closed. Memory therefore stays flat across the worker's lifetime — the **zero memory-drift** invariant (`ΔM = 0`).

Waffle ships **Igor-PHP** to verify `ΔM = 0` *statically*: it reports any service that mutates persistent state, forgets to reset a property, or touches a dangerous global, before that code reaches a resident worker. See the [runtime reference](../reference/runtime.md) for how to run it.
