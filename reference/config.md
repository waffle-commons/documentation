# Config Reference (`waffle-commons/config`)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *Beta-1 hardening retained: no process-env mutation*
> **Requires:** `ext-yaml` (the native PECL YAML extension)

Loads application configuration from YAML files using the native `ext-yaml` extension with `yaml.decode_php = 0` (no PHP-deserialisation gadgets). Environment overlays merge via `array_replace_recursive`; `%env(VAR_NAME)%` placeholders are resolved at load time against a **read-only env registry injected through the constructor** — never against `getenv()` or `$_ENV` directly.

## Constructor — exact signature

```php
namespace Waffle\Commons\Config;

use Waffle\Commons\Contracts\Config\ConfigInterface;
use Waffle\Commons\Contracts\Enum\Failsafe;

final class Config implements ConfigInterface
{
    /**
     * @param array<string, string> $env Read-only env registry consulted when
     *        resolving `%env(VAR)%` placeholders. Inject the merged DotEnv +
     *        process environment from `AppKernelFactory`. Defaults to an empty
     *        map — `%env(...)%` placeholders then resolve to `null`.
     */
    public function __construct(
        string $configDir,
        string $environment,
        Failsafe $failsafe = Failsafe::DISABLED,
        array $env = [],
    );
}
```

> **Beta 1 change.** The `array $env` parameter is new. Callers that previously relied on `getenv()` resolution must now pass an explicit map — typically `array_merge($dotenv->load(), getenv() ?: [])`. See [Environment registry](#environment-registry) below.

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
3. Resolves `%env(VAR)%` placeholders anywhere in the merged tree by reading the **injected `$env` registry** (constructor argument). Unknown keys resolve to `null`.

## Environment registry

Beta 1 removes all direct access to `getenv()` / `$_ENV` / `$_SERVER` from `Config` and `DotEnv` (the prior design used `putenv()` to inject `.env` values into the process environment — a thread-safety hazard under FrankenPHP worker mode, RFC: Roadmap Beta 1 Phase 0). The wiring contract is now explicit:

```php
// Canonical wiring (see AppKernelFactory in skeleton/workspace):
$processEnv  = getenv();                                // array<string,string>
$envRegistry = array_merge((new DotEnv($root))->load(), $processEnv);

$config = new Config(
    configDir:   $configDir,
    environment: $env,
    env:         $envRegistry,
);
```

### Precedence on collision

`array_merge` with string keys is **rightmost-wins**. In the canonical wiring above:

| Layer | Position in `array_merge` | Wins when both define the same key? |
| :--- | :--- | :--- |
| `.env` / `.env.local` (via `DotEnv::load()`) | left | no |
| OS / Docker / Kubernetes (via `getenv()`) | right | **yes** |

> **Example.** If your `.env` declares `APP_DEBUG=true` and the OS exports `APP_DEBUG=false`, the OS value wins — `$config->getBool('app.debug')` (via `'%env(APP_DEBUG)%'`) sees `false`. Restart your container or unset the OS variable to make the `.env` value take effect.

This matches the Twelve-Factor convention and the implicit precedence of the legacy DotEnv (which silently skipped any key already present in `$_ENV` / `$_SERVER`). You can invert it by flipping the merge order — e.g. `array_merge($processEnv, $dotenv->load())` — if your environment treats `.env` as source-of-truth.

### Type-normalization asymmetry

`DotEnv` normalizes the boolean-typed keys `APP_DEBUG` and `DEBUG` (anything in its `EXPECTED_TYPES` map): `true`/`yes`/`on`/`1` → `'1'`, `false`/`no`/`off`/`0` → `'0'`, anything else throws `InvalidArgumentException`. **`getenv()` performs no such normalization** — process-env values are passed through verbatim.

Consequence: a `.env` containing `APP_DEBUG=yes` produces `'1'`; an OS export of `APP_DEBUG=yes` produces `'yes'`. The YAML node `'%env(APP_DEBUG)%'` then resolves to a string that fails `Config::getBool('app.debug')` with `InvalidConfigurationException` — booleans in the YAML expect the literal YAML scalars `true` / `false`, not arbitrary strings.

**Best practice:** if you want process-env booleans to be safe under `getBool()`, normalize them at the wiring point — e.g. wrap `$processEnv` through your own `castBool()` before the merge, or expose booleans only via YAML literals rather than `%env()%` placeholders.

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
- `Waffle\Commons\Config\DotEnv` — **Beta 1**: pure `.env` / `.env.local` parser. `load(): array<string,string>` returns the parsed map; **no longer mutates** `putenv()`, `$_ENV`, or `$_SERVER`. Within DotEnv itself, the first file wins on key conflict (`.env` beats `.env.local`). Boolean-typed keys (`APP_DEBUG`, `DEBUG`) are validated + normalized to `'1'`/`'0'`; invalid values throw `InvalidArgumentException`.
- `Waffle\Commons\Config\Trait\ParserTrait` — shared parse helpers.
- `Waffle\Commons\Config\Exception\InvalidConfigurationException` — implements `InvalidConfigurationExceptionInterface`.
