<p align="center">
<img src="https://github.com/waffle-commons/.github/blob/main/assets/logo.png" alt="Waffle Ecosystem Logo">
</p>

# Waffle Framework Documentation

> **Strict, Secure, Fast.**

> [!WARNING]
> **BETA SOFTWARE**
> This version (`0.1.0-beta7`) is the **production surface** release: it builds what an API in production needs and the framework did not have yet. Four components arrive: `queue` (background jobs on Redis Streams, with a dead-letter store and a `queue:work` worker that drains on SIGTERM), `openapi` (an OpenAPI 3.1 document generated from `#[Route]` attributes and DTOs), `serializer` (compiled, reflection-free DTO normalizers and `Accept` negotiation) and `testing` (an in-process kernel test bridge with test doubles). Around them, a token-bucket rate limiter shared in Redis (NET-01); bounded timeouts, retries and a per-origin circuit breaker for outbound calls (NET-02/03); `/healthz` and `/readyz` probes, with readiness dropped before shutdown (OPS-01/02); reversible migrations with `db:status` and `db:rollback` (OPS-03); and dev profiler headers (DXP-01). The EcoShield Gateway alpha, built on this release, found a request-body framing bug in `http-client`, fixed here. The skeleton image moves to FrankenPHP 1.13.2 on PHP 8.5.12 for the PHP2Shell fixes. Mago stays at zero output with no baseline, and `wfl igor` stays a 0-KO gate. The four new components are not documented here yet (beta8 `[DOC-01]`). Beta-6's audit fixes and measured benchmark, Beta-5's runtime power (AOT, connection pooling, async deferral, reactive broadcast, telemetry, WebAuthn) and the Beta-1→4 security foundations all remain in place. Do not use in critical production environments without an independent security audit.

## 🎯 The Beta-7 contract

Waffle Beta-7 is built around two non-negotiable principles:

- **🧹 Zero-Debt.** Every component passes `vendor/bin/mago fmt`, `vendor/bin/mago lint`, `vendor/bin/mago analyze`, `vendor/bin/mago guard`, and `composer tests` with **zero errors, zero warnings, zero deprecations, zero infos, zero notices, zero helps, zero notes** — now including a repo-wide `cyclomatic-complexity` ceiling. No baseline files (`mago-*-baseline.toml`) exist anywhere in the tree — exceptions to the rule live as documented, reviewable `[analyzer.ignore]` entries in each component's `mago.toml`.
- **🐘 PHP 8.5 Strict.** Property Hooks for DTO validation, Asymmetric Visibility (`public private(set)`) for safe state exposure, typed constants for ecosystem-wide identifiers, `final readonly` for value objects, and `#[\Override]` on every interface implementation. The `mixed` type is forbidden in component surfaces (PSR-mandated exceptions aside).

`mago guard` enforces the **architectural perimeter** in every component: each `mago.toml` declares the exact list of permitted dependencies, and structural rules enforce `*Interface`, `*Exception`, and `Enum\` conventions across the whole ecosystem.

## Architecture

We follow the **Diátaxis** documentation framework to help you find exactly what you need.

| Quadrant | Goal | Description | Link |
| :--- | :--- | :--- | :--- |
| **Tutorials** | **Learning** | Step-by-step lessons to get you started. | [Start Here](tutorials/quick-start.md) |
| **How-To Guides** | **Problem Solving** | Practical recipes for specific tasks. | [Browse Guides](how-to/) |
| **Reference** | **Information** | Technical specifications of components. | [API Reference](reference/index.md) |
| **Explanation** | **Understanding** | Background, context, and design philosophy. | [Deep Dive](explanation/) |

## 🚀 Quick Navigation

### New to Waffle?
- [**Installation & First App**](tutorials/quick-start.md): Get up and running in 5 minutes with Docker.
- [**Your First Secured CRUD Endpoint**](tutorials/first-secured-crud-endpoint.md): Fail-closed ABAC, `#[Voter]`, CSRF, property-hook validation, and a pooled transactional write — step by step.
- [**First Steps with Async Work and Telemetry**](tutorials/first-async-and-telemetry.md): Defer post-response work, scrape `/waffle-metrics`, and read the memory/pool gauges.

### Solving a Problem?
- [**Secure Your Controller**](how-to/secure-a-controller.md): Security level, `#[Voter]`, `#[PublicAccess]`, CSRF.
- [**Authenticate Requests**](how-to/authentication.md): OAuth2/OIDC, JWT Bearer, gateway assertions, API keys — inbound + outbound (RFC-021).
- [**Configure CORS**](how-to/configure-cors.md): Fail-closed cross-origin policy, exact-origin allow-list, the `*`-with-credentials ban (SEC-04).
- [**Add Middleware**](how-to/middleware.md): Intercepting requests.
- [**Manage Configuration**](how-to/configuration.md): YAML + `%env(VAR)%` placeholders.
- [**Handle Errors**](how-to/error-handling.md): RFC 7807 JSON responses.
- [**Use Events**](how-to/events.md): PSR-14, `#[AsEventListener]`, lifecycle events.
- [**Routing**](how-to/routing.md): `#[Route]` and `#[Argument]`.
- [**Validate & Cleanse Input**](how-to/validate-input.md): `Assert` inside PHP 8.5 property hooks → RFC 7807 422.
- [**Run Database Migrations**](how-to/database-migrations.md): `waffle.database.*` config, then `make:migration`, `db:migrate`, `db:status` and `db:rollback` (RFC-022, OPS-03).
- [**Rate-Limit Requests**](how-to/rate-limit-requests.md): a token bucket per subject or client IP, shared in Redis, with `RateLimit-*` headers and `429` (NET-01).
- [**Make Outbound Calls Resilient**](how-to/make-outbound-calls-resilient.md): timeouts, bounded retries and a per-origin circuit breaker (NET-02/03).
- [**Run on Kubernetes**](how-to/run-on-kubernetes.md): `/healthz` and `/readyz`, draining before SIGTERM, migrations as a Job (OPS-01/02/03).
- [**Work on Multiple Components Locally**](how-to/local-development-workflow.md): `wfl link` / `unlink` / `debug` / `bench`.

### Need API Details?
- [**Security**](reference/security.md)
- [**Auth**](reference/auth.md)
- [**Routing**](reference/routing.md)
- [**Data & Persistence**](reference/data.md)
- [**`wfl` Developer CLI**](reference/wfl.md)
- [**Configuration Keys**](reference/configuration-keys.md)
- [**Index of Components**](reference/index.md)

### Under the Hood
- [**Architecture**](explanation/architecture.md): The Component-First philosophy, the contracts perimeter, and the statelessness mandate.
- [**Performance Strategy**](explanation/performance.md): Worker mode, preloading + AOT, memory-bounded proxying, pooling — and where the Beta-6 benchmark numbers will land.
- [**The Universal Data & Persistence Layer**](explanation/data-persistence.md): Why no ORM — SQR, per-backend compilers, stateless repositories, honest drivers (RFC-022).
- [**The Request Lifecycle**](explanation/lifecycle.md): From index.php to Response.
- [**Fail-Closed ABAC**](explanation/security-fail-closed-abac.md): Why missing voters now deny (Beta-1 / SEC-02).
- [**The Two Authorization Layers**](explanation/security-two-layer-authorization.md): Object-integrity Level ladder vs. context-aware voters — why there are two `analyze()` methods (Beta-5 / AUTHZ-01, ARCH-01).
- [**The Universal Authentication Bridge**](explanation/authentication-universal-bridge.md): One contract surface for every identity provider — and the zero-leak SecurityContext (RFC-021).
- [**CSRF: Signed Double-Submit**](explanation/security-csrf-double-submit.md): Stateless HMAC + per-browser binding (Beta-1 / SEC-01).

***

*Written for Waffle Framework 0.1.0-beta7 on PHP 8.5 (the skeleton image runs 8.5.12). The pages were verified one by one against 0.1.0-beta6 and updated with each beta7 change; the four new components are documented in beta8 `[DOC-01]`.*

***

> [![Discord](https://img.shields.io/discord/755288001592033391?color=7289da&label=discord&logo=discord&style=for-the-badge)](https://discord.gg/eKgywnfXr2)<br />
> *Join the core team and contributors on Discord to shape the future of cloud-native PHP.*

