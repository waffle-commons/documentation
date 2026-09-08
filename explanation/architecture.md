# Architecture & Philosophy

Waffle is built on a **"Component-First"** philosophy: the framework is not one package but a monorepo of **21 autonomous components**, each an independent Git repository and Composer package that can be used, versioned, and released on its own. This page explains the design forces that hold the ecosystem together — the dependency perimeter, the statelessness mandate, the request flow, and the ahead-of-time build step.

## The dependency perimeter: contracts at the centre

Every component depends **only** on `waffle-commons/contracts` — the root interface package — plus, where shared pure-function helpers are needed, `waffle-commons/utils`, which itself requires nothing but contracts. Beyond those two edges (and the PSR interface packages they formalise), a component's `composer.json` names no sibling: `security` does not require `http`, `data` does not require `container`, `routing` does not require `pipeline`. Components collaborate exclusively through the interfaces in `Waffle\Commons\Contracts\*`; the **application** (your `AppKernelFactory`) is the only place concretes meet.

This is not a convention that relies on discipline — it is machine-enforced. Each component's `mago.toml` declares its exact permitted dependency list, and `mago guard` fails the build on any undeclared edge. The same guard enforces the structural naming rules ecosystem-wide: every interface is `*Interface`, every exception ends in `*Exception`, every enum lives in an `Enum\` namespace.

A small number of components exist precisely to be the *sole importer* of a third-party SDK, so the rest of the ecosystem never sees it:

- `cache` is the only importer of `predis` (its `ArrayCache` / `FileCache` adapters need nothing);
- `auth` is the only importer of `web-auth/webauthn-lib`, confined behind `WebAuthnVerifierInterface`;
- `telemetry-otel` is the only importer of the OpenTelemetry SDK — the framework core speaks the no-op tracing contracts and pays nothing until the bridge is wired (see [Contract-First Observability](observability-telemetry.md)).

The payoff of the perimeter is substitution: any component can be replaced by anything else that honours the contracts, and a security fix in one component ships without a coordinated release of the others.

## The 21 components, by role

| Group | Components | Role |
| :--- | :--- | :--- |
| **Foundation** | `contracts` · `utils` · `config` · `container` | The interface hub; pure-function helpers (`Assert`, reflection tooling); native YAML + DotEnv configuration; PSR-11 autowiring container with worker-mode `reset()`. |
| **HTTP** | `http` · `http-client` · `routing` · `pipeline` | PSR-7/17 messages and emitter; memory-bounded PSR-18 cURL client with default-on SSRF pinning and concurrent fan-out; `#[Route]` attribute router; PSR-15 middleware stack. |
| **Platform services** | `event-dispatcher` · `log` · `error-handler` · `cache` | PSR-14 events with `#[AsEventListener]` discovery; PSR-3 JSON stream logger; RFC 7807 error rendering; PSR-6/16 caching with stampede protection. |
| **Security & identity** | `security` · `auth` | Fail-closed ABAC (`#[Voter]` / `#[PublicAccess]`), stateless HMAC CSRF, fail-closed CORS; the Universal Authentication Bridge (RFC-021) — OAuth2/OIDC, JWT, gateway assertions, API keys, WebAuthn passkeys. |
| **Data** | `data` | The Universal Data & Persistence Layer (RFC-022): backend-agnostic query AST, per-engine compilers, stateless repositories, memory-resident connection pools, forward-only migrations. |
| **CLI** | `console` | Zero-magic command runtime: `db:migrate`, `route:list`, the `make:*` scaffolding makers, and the AOT compilers. |
| **Runtime & kernel** | `runtime` · `waffle` | The FrankenPHP worker loop (classic SAPI fallback); the kernel facade — `AbstractKernel`, controller dispatch, argument resolution, `#[Broadcast]` reactive hooks. |
| **Observability** | `telemetry` · `telemetry-otel` | SDK-free Prometheus metrics behind the fail-closed `/waffle-metrics` endpoint; the opt-in OpenTelemetry tracing bridge. |
| **Async** | `async` | Fiber-based finish-request task deferral under a bounded per-request budget. |

The [component reference index](../reference/index.md) links each one to its full API page.

## The statelessness mandate

Waffle targets **FrankenPHP worker mode**: the process boots once, then serves thousands of requests without dying. That inverts the classic PHP safety model — nothing is thrown away for free at the end of a request anymore. Whatever a request leaves mutated on a shared service is *inherited by the next request*, which is both a correctness bug (state bleed) and a security bug (another user's data still in memory).

The mandate that follows is simple to state and ruthless to enforce: **every piece of request-scoped state must be explicitly recyclable**. Services that hold such state implement `ResettableInterface`; at the end of each worker iteration the runtime cascades `reset()` through the container — the async task queue empties, the broadcast buffer clears, pooled connections roll back stragglers, the security context drops its identity. Memory stays flat across the worker's lifetime: the **zero memory-drift** invariant, `ΔM = 0`.

Compliance is audited, not assumed. The **igor-php** static audit (`wfl igor`, `igor:audit`) flags any class that mutates persistent state, forgets a property in its `reset()`, or touches a dangerous global — and the ecosystem gate is **0 KO**. The flip side of the mandate is what makes worker mode worth it: state that is *deliberately* memory-resident (warm PDO handles in the [connection pool](connection-pooling.md), the compiled route table, APCu metric counters) survives across requests by design, with its reset semantics spelled out.

## The request flow: kernel + pipeline

A request enters through `WaffleRuntime`, which rebuilds a PSR-7 request from the worker's globals and hands it to the kernel. The kernel pushes it through a PSR-15 `MiddlewareStack` in a canonical order — error handling first, then telemetry, host/CORS gates, authentication, routing, CSRF, ABAC authorization, transaction isolation, secure headers — until the terminal dispatcher resolves the controller through the `SecureContainer` and invokes it. The response flows back out through the same stack. The step-by-step walk-through lives in [The Request Lifecycle](lifecycle.md); the security ordering rationale in [How-To: Secure a Controller](../how-to/secure-a-controller.md).

The kernel is **event-driven** (PSR-14) around that spine. Lifecycle events allow request injection and response decoration without touching core logic, and the post-emission `TerminateEvent` — fired *after* the response has been flushed to the client — is the anchor for everything that should never sit on the user-perceived latency path:

- [finish-request task deferral](async-finish-request-deferral.md) drains the bounded per-request queue of deferred work (mail, audit, webhooks), each task isolated in its own Fiber;
- the [reactive broadcast flush](reactive-broadcast.md) pushes recorded `#[Broadcast]` mutations over Server-Sent Events;
- in dev, the orphaned-connection listener inspects the connection tracker and warns about handles a request failed to release.

## Ahead-of-Time compilation

The first request a fresh worker serves pays the full bootstrap bill: reflection-based container wiring and `#[Route]` discovery. Both are deterministic per build, so Beta-5 moves them into an explicit build step (RFC-019): `container:compile` emits a reflection-free compiled container proven graph-identical to the runtime one, and `route:compile` serialises an O(depth) route trie. Both artifacts are consulted only when the operator sets `WAFFLE_AOT=1`, and a missing or corrupt artifact degrades to the reflection path with a logged warning — the fast path can only ever be *faster*, never *different*. See [Ahead-of-Time Compilation](aot-compilation.md), and the broader [Performance Strategy](performance.md) it belongs to.

## What holds it together

Three invariants recur in every design decision above, and they are the shortest honest summary of the architecture:

1. **Contracts-only edges** — components meet through interfaces, enforced by `mago guard`; SDKs are quarantined in their sole-importer components.
2. **Stateless across requests** — request-scoped state is resettable and audited (`wfl igor` 0 KO); memory-resident state is deliberate and reset-aware.
3. **Nothing magic at runtime** — wiring is explicit in the application factory, AOT is an opt-in build step, and fail-closed is the default posture (ABAC, CORS, CSRF, the metrics endpoint).

> *Verified for Waffle Framework 0.1.0-beta6 running on PHP 8.5.6+.*
