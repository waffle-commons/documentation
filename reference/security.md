# Security Reference (`waffle-commons/security`)

The Security component enforces Attribute-Based Access Control (ABAC) via the Container.

## `SecureContainer`

The `SecureContainer` is a decorator that wraps the inner PSR-11 container.
- It intercepts every `get()` call.
- It passes the resolved object to the Security Analyzer.
- If validation fails, it throws a `SecurityException`.

## Security Rules (Levels)

Security is configured via an integer level (1-10) in `app.yaml`.

- **Level 1 (`Level1Rule`)**: Consistency Check. Ensures the object is an instance of its own class.
- **Level 2 - 9**: Intermediate validation rules (e.g., Code Integrity, Property Type safety).
- **Level 10 (`Level10Rule`)**: Paranoid Check. Maximum strictness used for High Security environments.

> **Note**: Specific `#[Rule]` attributes for granular control are planned for future releases.
