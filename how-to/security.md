# How to Secure Your Application

Waffle implements a global **Attribute-Based Access Control (ABAC)** system integrated into the Container.

## Configuring the Security Level

The security level defines the rigorousness of checks performed on every object instantiated by the application. You can configure this in your `config/app.yaml`.

```yaml
waffle:
  security:
    # Level 1 (Basic) to Level 10 (Paranoid)
    level: 5
```

### What do the levels do?
Each level adds a stronger rule to the validation chain.
- **Level 1**: Ensures objects are instances of their own class (Basic Integrity).
- **Levels 2-10**: Perform increasingly strict checks on object state and properties.

## Using the Secure Container

You don't need to do anything special to use the Secure Container; it is automatically wrapped around the core container during the System boot.

Any service you inject into your controller has already passed the security checks.

```php
public function __construct(private UserService $service)
{
    // If we are here, $service has been analyzed and approved by the Security component.
}
```

## Granular Control (Planned)

*Support for applying specific security rules via attributes on individual Controllers (e.g., `#[Rule]`) is planned for a future release.*
