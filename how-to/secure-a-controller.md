# How to Secure a Controller

In Waffle v0.1.0-alpha4, security is enforced globally via the **Security Level** in your configuration. This ensures that *every* controller and service instantiated by the container adheres to strict integrity rules.

## 1. Configure the Security Level

Open your `config/app.yaml` file:

```yaml
waffle:
  security:
    # Set to a high level (e.g., 10) for maximum security
    level: 10
```

## 2. Granular Control (Planned)

In upcoming releases, you will be able to apply specific rules to individual controllers or methods using the `#[Rule]` attribute.

> **Planned Feature Preview:**
>
> ```php
> use Waffle\Commons\Security\Attribute\Rule;
> use Waffle\Commons\Security\Rule\Level10Rule;
> 
> #[Rule(Level10Rule::class)]
> class AdminController
> {
>     // ...
> }
> ```

For now, rely on `waffle.security.level` to secure your application.
