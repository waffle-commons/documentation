# Config Reference (`waffle-commons/config`)

> **Release:** `v0.1.0-beta0`
> **Requires:** `ext-yaml` (the native PECL YAML extension)

Loads application configuration from YAML files using the native `ext-yaml` extension with `yaml.decode_php = 0` (no PHP-deserialisation gadgets). Environment overlays merge via `array_replace_recursive`; `%env(VAR_NAME)%` placeholders resolve at load time.

## Constructor — exact signature

```php
namespace Waffle\Commons\Config;

use Waffle\Commons\Contracts\Config\ConfigInterface;
use Waffle\Commons\Contracts\Enum\Failsafe;

final class Config implements ConfigInterface
{
    public function __construct(
        string $configDir,
        string $environment,
        Failsafe $failsafe = Failsafe::DISABLED,
    );
}
```

## Typed getters (PSR-style)

All four getters share the same shape: `(string $key, ?T $default = null): ?T`. Mismatched types throw `InvalidConfigurationException` (which implements `InvalidConfigurationExceptionInterface`).

```php
public function getInt(string $key, ?int $default = null): ?int;
public function getString(string $key, ?string $default = null): ?string;
public function getArray(string $key, ?array $default = null): ?array;
public function getBool(string $key, ?bool $default = null): ?bool;
```

Keys are dot-paths. Example:

```php
$level = $config->getInt('waffle.security.level', default: 1);     // 1 .. 10
$paths = $config->getArray('waffle.trusted_hosts', default: []);
$debug = $config->getBool('app.debug', default: false);
```

## File layout

```
config/
├── app.yaml           # base, always loaded if it exists
├── app_dev.yaml       # environment overlay
├── app_prod.yaml      # environment overlay
└── app_test.yaml      # environment overlay
```

`Config::loadConfigurationFiles()`:

1. Loads `app.yaml` if it exists.
2. Loads `app_{environment}.yaml` if it exists; merges on top via `array_replace_recursive`.
3. Resolves `%env(VAR)%` placeholders anywhere in the merged tree by reading `getenv()`.

## Failsafe mode

```yaml
# No app.yaml needed — Failsafe::ENABLED seeds:
waffle:
  security:
    level: 1
```

Pass `Failsafe::ENABLED` to skip filesystem loading entirely and use the safe baseline. Used by the `ErrorHandlerMiddleware` boot path so a totally broken config still allows the error renderer to function.

## Sibling classes

- `Waffle\Commons\Config\YamlParser` — `final` wrapper around `yaml_parse_file()`.
- `Waffle\Commons\Config\DotEnv` — `.env` file loader writing into `getenv()`.
- `Waffle\Commons\Config\Trait\ParserTrait` — shared parse helpers.
- `Waffle\Commons\Config\Exception\InvalidConfigurationException` — implements `InvalidConfigurationExceptionInterface`.
