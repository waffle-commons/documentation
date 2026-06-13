<p align="center">
<img src="https://github.com/waffle-commons/.github/blob/main/assets/logo.png" alt="Waffle Ecosystem Logo">
</p>

# Waffle Framework Documentation

> **Strict, Secure, Fast.**

> [!WARNING]
> **BETA SOFTWARE**
> This version (`0.1.0-beta4`) is the **security-hardening & worker-mode-stability** release — the RC-readiness groundwork on top of Beta-3's identity-federation & stateless-persistence wave. Beta-4 closes the core security gaps (session-fixation rotation + cryptographic CSRF binding, **default-on** SSRF resolve→validate→pin with an internal allowlist, timing-safe comparison enforcement, **fail-closed CORS**, and path-traversal guardrails), hardens the architecture for resident-worker mode (typed kernel lifecycle events, stream-resource ownership, an interface-based response converter, a standalone uploaded-files normalizer), and adds worker-mode **diagnostics** (a boot-time state-reset compliance scanner and a dev-only orphaned-connection tracer for PDO/Redis/streams) plus developer-experience tooling (`wfl check:all` / `monorepo:sync`, native `mb_trim`, a mockable `ValidatorInterface`); the `wfl igor` worker-safety audit stays a 0-KO gate. Beta-3's `auth` (Universal Authentication Bridge, RFC-021) and `data` (Universal Data & Persistence Layer, RFC-022) components — and the Beta-1/Beta-2 foundations (fail-closed ABAC, stateless HMAC CSRF bound to `WAFFLE_SID`, typed `405`/`404` contracts) — all remain in place. Do not use in critical production environments without an independent security audit.

## 🎯 The Beta-4 contract

Waffle Beta-4 is built around two non-negotiable principles:

- **🧹 Zero-Debt.** Every component passes `vendor/bin/mago fmt`, `vendor/bin/mago lint`, `vendor/bin/mago analyze`, `vendor/bin/mago guard`, and `composer tests` with **zero errors, zero warnings, zero deprecations, zero infos, zero notices, zero helps, zero notes**. No baseline files (`mago-*-baseline.toml`) exist anywhere in the tree — exceptions to the rule live as documented, reviewable `[analyzer.ignore]` entries in each component's `mago.toml`.
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

### Solving a Problem?
- [**Secure Your Controller**](how-to/secure-a-controller.md): Security level, `#[Rule]`, `#[Voter]`, CSRF.
- [**Authenticate Requests**](how-to/authentication.md): OAuth2/OIDC, JWT Bearer, gateway assertions, API keys — inbound + outbound (RFC-021).
- [**Add Middleware**](how-to/middleware.md): Intercepting requests.
- [**Manage Configuration**](how-to/configuration.md): YAML + `%env(VAR)%` placeholders.
- [**Handle Errors**](how-to/error-handling.md): RFC 7807 JSON responses.
- [**Use Events**](how-to/events.md): PSR-14, `#[AsEventListener]`, lifecycle events.
- [**Routing**](how-to/routing.md): `#[Route]` and `#[Argument]`.
- [**Validate & Cleanse Input**](how-to/validate-input.md): `Assert` inside PHP 8.5 property hooks → RFC 7807 422.
- [**Run Database Migrations**](how-to/database-migrations.md): `waffle.database.*` config + `bin/waffle db:migrate` (RFC-022).
- [**Work on Multiple Components Locally**](how-to/local-development-workflow.md): `wfl link` / `unlink` / `debug` / `bench`.

### Need API Details?
- [**Security**](reference/security.md)
- [**Auth**](reference/auth.md)
- [**Routing**](reference/routing.md)
- [**Data & Persistence**](reference/data.md)
- [**`wfl` Developer CLI**](reference/wfl.md)
- [**Index of Components**](reference/index.md)

### Under the Hood
- [**Architecture**](explanation/architecture.md): The Component-First philosophy.
- [**The Universal Data & Persistence Layer**](explanation/data-persistence.md): Why no ORM — SQR, per-backend compilers, stateless repositories, honest drivers (RFC-022).
- [**The Request Lifecycle**](explanation/lifecycle.md): From index.php to Response.
- [**Fail-Closed ABAC**](explanation/security-fail-closed-abac.md): Why missing voters now deny (Beta-1 / SEC-02).
- [**The Universal Authentication Bridge**](explanation/authentication-universal-bridge.md): One contract surface for every identity provider — and the zero-leak SecurityContext (RFC-021).
- [**CSRF: Signed Double-Submit**](explanation/security-csrf-double-submit.md): Stateless HMAC + per-browser binding (Beta-1 / SEC-01).

***

*Verified for Waffle Framework 0.1.0-beta4 running on PHP 8.5.5+.*

***

> [![Discord](https://img.shields.io/discord/755288001592033391?color=7289da&label=discord&logo=discord&style=for-the-badge)](https://discord.gg/eKgywnfXr2)<br />
> *Join the core team and contributors on Discord to shape the future of cloud-native PHP.*

