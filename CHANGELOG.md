# Changelog — waffle-commons/documentation

All notable changes to this repository are documented in this file.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
Released in lockstep with the Waffle Commons umbrella tag.

> This repository carries no PHP package — it is the canonical Diátaxis documentation tree,
> tagged alongside the framework so every release has a matching documentation snapshot.
> Entries begin at `0.1.0-beta6`, when this changelog was introduced; earlier tags
> (`0.1.0-alpha5` → `0.1.0-beta5`) exist but were not accompanied by release notes here.

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
