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
- **Post-Emission Tasks**: The `TerminateEvent` allows for heavy asynchronous processing (e.g., sending emails, cache warming) after the response has been sent to the client, maximizing performance in worker modes (FrankenPHP/RoadRunner). Two Beta-5 features build directly on this hook: [finish-request deferral](async-finish-request-deferral.md) (a single-thread Fiber isolation boundary, not background threads, that drains a bounded per-request task budget) and the [reactive broadcast flush](reactive-broadcast.md) (the SSE push of recorded `#[Broadcast]` mutations). Memory-resident state that stays warm across requests but is reset each iteration — the persistence-layer example being [memory-resident connection pooling](connection-pooling.md) — also rides this worker-mode model.

This structure ensures maximum extensibility without modifying the core framework logic.

## Contract-First Observability

The dependency perimeter (enforced by `mago guard`) keeps observability SDK-free: the framework core imports only `Waffle\Commons\Contracts\Telemetry\*` and their no-op defaults, while `waffle-commons/telemetry-otel` is the *sole* OpenTelemetry-SDK importer and `waffle-commons/telemetry` exposes SDK-free Prometheus metrics. See [Contract-First Observability](observability-telemetry.md).

## Authentication: WebAuthn / Passkeys

Passkeys are an *inbound scheme* of the RFC-021 Universal Authentication Bridge — not a separate subsystem. A single audited adapter sits behind the WebAuthn contracts, with the only state (challenge store + credential repository) pushed out into app storage so the adapter stays igor-clean. See [WebAuthn & Passkeys](webauthn-passkeys.md).

## Ahead-of-Time Compilation

Build-time AOT moves deterministic container wiring and route discovery out of the first-request path: the compiled container composes the reflection container (graph-identical) and a route trie replaces the sequential matcher, both opt-in via `WAFFLE_AOT=1`. See [Ahead-of-Time Compilation](aot-compilation.md).
