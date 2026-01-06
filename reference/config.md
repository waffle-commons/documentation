# Config Reference (`waffle-commons/config`)

The Config component handles loading application configuration from YAML files and environment variables.

## Usage

The `Waffle\Commons\Config\Config` class is responsible for loading `app.yaml` and environment-specific overrides (e.g., `app_prod.yaml`).

### Loading Configuration

```php
use Waffle\Commons\Config\Config;

$config = new Config(
    configDir: __DIR__ . '/../config',
    environment: 'prod'
);
```

### Retrieving Values

The `Config` class provides strictly typed methods to retrieve configuration values.

#### `getInt(string $key, ?int $default = null): ?int`

Retrieves an integer value. Throws `InvalidConfigurationException` if the value exists but is not an integer.

```php
$level = $config->getInt('waffle.security.level', 1);
```

#### `getString(string $key, ?string $default = null): ?string`

Retrieves a string value.

```php
$path = $config->getString('waffle.paths.controllers');
```

#### `getBool(string $key, bool $default = false): bool`

Retrieves a boolean value.

```php
$debug = $config->getBool('waffle.debug', false);
```

#### `getArray(string $key, ?array $default = null): ?array`

Retrieves an array.

```php
$settings = $config->getArray('waffle.modules');
```

## Structure

Configuration keys are dot-notated. For example, `waffle.security.level` corresponds to:

```yaml
waffle:
  security:
    level: 1
```

## Environment Variables

You can reference environment variables in your YAML files using the `%env(VAR_NAME)%` syntax.

```yaml
database:
  password: '%env(DB_PASSWORD)%'
```
