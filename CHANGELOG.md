# Changelog — waffle-commons/documentation

All notable changes to this repository are documented in this file.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
Released in lockstep with the Waffle Commons umbrella tag.

> This repository carries no PHP package — it is the canonical Diátaxis documentation tree,
> tagged alongside the framework so every release has a matching documentation snapshot.
> Entries begin at `0.1.0-beta6`, when this changelog was introduced; earlier tags
> (`0.1.0-alpha5` → `0.1.0-beta5`) exist but were not accompanied by release notes here.

## [Unreleased]

### Added
- Three how-to guides: `how-to/rate-limit-requests.md` (NET-01), `how-to/make-outbound-calls-resilient.md`
  (NET-02/03) and `how-to/run-on-kubernetes.md` (probes, draining, migrations as a Job — OPS-01/02/03).
- `reference/configuration-keys.md`: every `waffle.*` key the skeleton declares, with its type, template
  value, default and reader. It flags `waffle.debug`, `waffle.log.channel` and `waffle.log.stream` as
  declared but read by nothing, and the `waffle.paths.event_listeners` default as pointing at a directory
  the skeleton does not have.
- Reference sections for the NET/OPS surface: `contracts.md` (rate limiting, outbound resilience, health
  checks, `ClientIpResolverInterface`), `security.md` (`RateLimitMiddleware`, `RateLimiter`,
  `StorageFailureMode`), `http.md` (`TrustedProxyClientIpResolver`), `pipeline.md` (`ProbeMiddleware`,
  `ReadinessProbe`), `data.md` (the Redis state storages, `ConnectionPoolHealthCheck`) and `cache.md`
  (`CacheHealthCheck`); a Beta-7 section in `reference/index.md`.

### Changed
- `reference/http-client.md`: `Client` implements `SettlingClientInterface` and takes its timeouts from a
  `TimeoutPolicy` (`withTimeouts()`): they are no longer hardcoded ceilings. New *Resilience* section
  (`RetryingClient`, `CircuitBreakerClient`, `CircuitObservers`) and `CircuitOpenException` in the exception
  table.
- `reference/http-client.md`: a host the SSRF guard cannot resolve is a `NetworkException` (cURL code 6), no longer an `SsrfException`.
- `reference/auth.md`: `Client\AuthenticatedSettlingClient`, the outbound credentials decorator that keeps a batch concurrent.
- `how-to/database-migrations.md`: migrations run from the skeleton's production image, which now ships `bin/` and `migrations/`.
- `how-to/database-migrations.md` covers the whole migration cycle — `make:migration`, `db:migrate`, `db:status`,
  `db:rollback` — with each command's exit codes, and the transactional-DDL caveat now names MariaDB, Oracle and
  SQL Server too (OPS-03).
- `reference/console.md`, `reference/data.md` and `reference/contracts.md` document the migration commands,
  `MigrationRunner::rollback()` and `status()`, `MigrationLedger`, `MigrationState` and `MigrationStatus` (OPS-03).
- `reference/contracts.md` shows `KernelInterface::$container` (CARRY-04).

## [0.1.0-beta6] — 2026-09

**Theme: the documentation is made to describe the code that actually exists (AXE 3).**

### Added
- Two tutorials, restoring quadrant balance: **Your First Secured CRUD Endpoint** and
  **First Steps with Async Work and Telemetry**.
- A **Beta-6 additions** section in `reference/index.md` covering the two contracts-surface
  changes from the audit remediation (`SubjectResolverInterface`; `#[PublicAccess]` restricted
  to `Attribute::TARGET_METHOD`).

### Changed
- `explanation/architecture.md` rewritten from stale Beta-1/2-era content to the real
  21-component ecosystem, grounded in a `composer.json` sweep of all 21.
- `explanation/performance.md` de-orphaned (3 inbound links) and made the home of the measured
  AXE 5 benchmark numbers — including the reframing of the retired "5–10× RAM vs PHP-FPM"
  claim as a growth *slope* (memory grows 10.5× slower per concurrent request).
- Every reference page verified symbol-by-symbol against the live public surface — roughly 30
  stale claims corrected (pre-ARCH-03 kernel API, beta5 pool signatures, missing beta6
  hardenings).

### Fixed
- A **`#[Rule]` attribute documented across six pages that has never existed** in the codebase,
  swept corpus-wide.
- Duplicate `how-to/security.md` merged into `how-to/secure-a-controller.md` and deleted.
- Link graph checked in full: **331 links, 0 broken, 0 anchor mismatches**.
