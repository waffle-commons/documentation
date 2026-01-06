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
- `waffle-commons/http`: PSR-7/17 implementation.
- `waffle-commons/config`: YAML & DotEnv loader.
- `waffle-commons/error-handler`: RFC 7807 Error Renderer.

This structure allows you to compose your application with exactly what you need.
