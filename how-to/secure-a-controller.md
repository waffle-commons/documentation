# How-To: Secure a Controller

Waffle Framework enforces security through a **Global Security Level** system. Instead of securing individual controllers with attributes, you define a security baseline for your entire application. The framework ensures all instantiated objects, including controllers, adhere to this level.

## Configuration

The security level is configured in your `config/app.yaml` file under the `waffle.security.level` key.

```yaml
waffle:
  security:
    level: 1 # Levels 1-10
```

## Security Levels

The security monitoring is performed by the `SecureContainer` when retrieving services or controllers.

- **Level 1 (Basic)**: Validates object integrity. Ensures that the instantiated object is actually an instance of its declared class.
- **Level 2-10 (Advanced)**: Applies progressively stricter auditing rules. (See `Waffle\Commons\Security\Rule` namespace for implementation details of each level).

## How It Works

1.  **Request**: When a request comes in, the `ControllerDispatcher` requests the controller from the `SecureContainer`.
2.  **Analysis**: The `SecureContainer` intercepts this request and passes the object to the `Security` service.
3.  **Enforcement**: The `Security` service runs the configured `LevelXRule` checks on the controller instance.
4.  **Result**:
    - If valid, the controller is returned and executed.
    - If invalid, a `SecurityException` is thrown, returning a 500 error (or handled by your Error Handler).

> [!NOTE]
> Currently, per-controller security attributes (like `#[Rule]`) are not supported. Security is enforced globally to ensure a consistent security posture across the entire application.
