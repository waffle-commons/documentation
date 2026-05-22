<p align="center">
<img src="https://github.com/waffle-commons/.github/blob/main/assets/logo.png" alt="Waffle Ecosystem Logo">
</p>

# Waffle Framework Documentation

> **Strict, Secure, Fast.**

> [!WARNING]
> **BETA SOFTWARE**
> This version (`v0.1.0-beta1`) is the security-hardening release on top of Beta 0. It lands the MUST-HAVE items from the Beta-0 audit: fail-closed ABAC with a new `#[PublicAccess]` opt-out, stateless HMAC CSRF tokens bound to a per-browser `WAFFLE_SID` cookie, an SSRF allowlist on the HTTP client, removal of the static `GlobalsFactory` for FrankenPHP safety, and a proper typed `RouteNotFoundException` for 404 mapping. The `CsrfTokenManagerInterface` signature changed — see [contracts.md](reference/contracts.md). Do not use in critical production environments without an independent security audit.

## 🎯 The Beta-1 contract

Waffle Beta-1 is built around two non-negotiable principles:

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
- [**Add Middleware**](how-to/middleware.md): Intercepting requests.
- [**Manage Configuration**](how-to/configuration.md): YAML + `%env(VAR)%` placeholders.
- [**Handle Errors**](how-to/error-handling.md): RFC 7807 JSON responses.
- [**Use Events**](how-to/events.md): PSR-14, `#[AsEventListener]`, lifecycle events.
- [**Routing**](how-to/routing.md): `#[Route]` and `#[Argument]`.

### Need API Details?
- [**Security**](reference/security.md)
- [**Routing**](reference/routing.md)
- [**Index of Components**](reference/index.md)

### Under the Hood
- [**Architecture**](explanation/architecture.md): The Component-First philosophy.
- [**The Request Lifecycle**](explanation/lifecycle.md): From index.php to Response.
- [**Fail-Closed ABAC**](explanation/security-fail-closed-abac.md): Why missing voters now deny (Beta-1 / SEC-02).
- [**CSRF: Signed Double-Submit**](explanation/security-csrf-double-submit.md): Stateless HMAC + per-browser binding (Beta-1 / SEC-01).

---

*Verified for Waffle Framework v0.1.0-beta1 running on PHP 8.5.5+.*
