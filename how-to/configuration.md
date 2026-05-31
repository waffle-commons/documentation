# How to Manage Configuration

Waffle allows you to manage settings using YAML files and Environment variables.

## Adding a Configuration Key

Open `config/app.yaml` and add your custom configuration.

```yaml
# config/app.yaml
myapp:
  api_key: '%env(MY_API_KEY)%'
  timeout: 30
```

`%env(VAR)%` placeholders are resolved at boot time against an **injected env registry** (`Config`'s fourth constructor argument). The registry is built by `AppKernelFactory` by merging the parsed `.env` file with the live process environment — see [Environment variables: how the merge works](#environment-variables-how-the-merge-works) below for the precedence rules.

### Database configuration (RFC-022)

The `waffle-commons/data` layer reads its connection settings from the `waffle.database` block. Credentials come from `DB_*` env vars; `migrations_path` (relative to the project root) tells `bin/waffle db:migrate` where to find SQL scripts:

```yaml
# config/app.yaml
waffle:
  database:
    driver: 'mysql'
    host: '%env(DB_HOST)%'
    port: '%env(DB_PORT)%'
    database: '%env(DB_NAME)%'
    username: '%env(DB_USER)%'
    password: '%env(DB_PASSWORD)%'
    charset: 'utf8mb4'
    migrations_path: 'migrations'
```

`AppKernelFactory::buildConnectionPool($config)` turns this into a `PDOConnectionPool`. See [How to: Database Migrations](database-migrations.md) and the [data reference](../reference/data.md).

## Injecting Configuration into a Service

You can inject the core `ConfigInterface` into any service to access these values.

```php
namespace App\Service;

use Waffle\Commons\Contracts\Config\ConfigInterface;

class ApiClient
{
    private string $apiKey;

    public function __construct(ConfigInterface $config)
    {
        // Enforce type safety
        $this->apiKey = $config->getString('myapp.api_key');
        
        // Optional with default value
        $timeout = $config->getInt('myapp.timeout', 5);
    }
}
```

The Container will automatically inject the active configuration instance when `ApiClient` is requested.

## Environment variables: how the merge works

> **Beta 1 change.** `DotEnv` no longer mutates the global PHP environment (no `putenv()`, no writes to `$_ENV` / `$_SERVER`). Instead, `AppKernelFactory` reads the `.env` file via `DotEnv::load()`, merges it with `getenv()`, and injects the result into `Config`. This is required for FrankenPHP worker-mode safety (worker mode shares process state across requests; the legacy `putenv()` was not thread-safe).

The canonical wiring (already in `skeleton` and `workspace` `AppKernelFactory`):

```php
$processEnv  = getenv();                                // OS / Docker / Kubernetes
$envRegistry = array_merge((new DotEnv($root))->load(), $processEnv);

$config = new Config(
    configDir:   $rootConfig,
    environment: $env,
    env:         $envRegistry,
);
```

### Who wins when both define the same key?

`array_merge` with string keys is **rightmost-wins**, so the process environment beats `.env`:

| Source | Position in `array_merge` | Wins? |
| :--- | :--- | :--- |
| `.env` / `.env.local` | left | no |
| OS / Docker / Kubernetes (`getenv()`) | right | **yes** |

#### Concrete example

| Source | Declares | Result |
| :--- | :--- | :--- |
| `.env` | `APP_DEBUG=true` (normalized to `'1'`) | overridden |
| Docker `environment:` | `APP_DEBUG=false` | **kept** — `$envRegistry['APP_DEBUG'] === 'false'` |

If you just edited `.env` and it doesn't appear to take effect, **check whether the same variable is exported by your shell or Docker compose file** — the OS export will silently win.

To make `.env` source-of-truth instead, flip the merge order in your `AppKernelFactory`:

```php
$envRegistry = array_merge($processEnv, (new DotEnv($root))->load()); // .env wins
```

### Type-normalization asymmetry (foot-gun)

`DotEnv` validates and normalizes `APP_DEBUG` / `DEBUG`:

- `APP_DEBUG=true` / `yes` / `on` / `1` → stored as `'1'`
- `APP_DEBUG=false` / `no` / `off` / `0` → stored as `'0'`
- Anything else (e.g. `APP_DEBUG=maybe`) → `InvalidArgumentException`

`getenv()` does **not** perform this normalization — it returns whatever the OS gave us, character-for-character. So:

| Source | Value | After DotEnv | After getenv() |
| :--- | :--- | :--- | :--- |
| `.env` | `APP_DEBUG=yes` | `'1'` | — |
| OS export | `APP_DEBUG=yes` | — | `'yes'` |

When the YAML uses `'%env(APP_DEBUG)%'` and `Config::getBool('app.debug')` is called, the second case throws `InvalidConfigurationException` because `'yes'` is a string, not a YAML boolean.

**Safe patterns:**
- Always export OS env values in the canonical form (`true` / `false` or `1` / `0`).
- Or normalize `$processEnv` yourself before merging.
- Or expose booleans in YAML as literals (`debug: true`) instead of `%env()%` placeholders.
