# Architecture & Philosophy

Waffle is built on a **"Micro-Component"** philosophy.

## The Decoupled Ecosystem

Unlike monolithic frameworks, Waffle is composed of standalone libraries that can be used independently.

### Core Packages
- `waffle-commons/waffle`: The Kernel and Bootstrapper.
- `waffle-commons/runtime`: The Application Entry Point (FrankenPHP support).
- `waffle-commons/contracts`: Shared Interfaces.

### Functional Packages
- `waffle-commons/pipeline`: PSR-15 Middleware Stack.
- `waffle-commons/routing`: Attribute-based Router.
- `waffle-commons/security`: ABAC Security System.
- `waffle-commons/container`: PSR-11 Dependency Injection.
- `waffle-commons/event-dispatcher`: PSR-14 Event System.
- `waffle-commons/log`: PSR-3 JSON Stream Logger.
- `waffle-commons/http`: PSR-7/17 implementation.
- `waffle-commons/config`: YAML & DotEnv loader.
- `waffle-commons/error-handler`: RFC 7807 Error Renderer.

## Event-Driven Core

Starting with Alpha 5, the Waffle Kernel is **event-driven**. It leverages PSR-14 to provide hooks throughout the request/response lifecycle:

- **Request Injection**: Modify the request before it enters the middleware pipeline.
- **Response Decoration**: Standardize or mutate responses after processing.
- **Post-Emission Tasks**: The `TerminateEvent` allows for heavy asynchronous processing (e.g., sending emails, cache warming) after the response has been sent to the client, maximizing performance in worker modes (FrankenPHP/RoadRunner).

This structure ensures maximum extensibility without modifying the core framework logic.
