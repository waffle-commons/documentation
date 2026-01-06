# Config Reference (`waffle-commons/config`)

Manages configuration loading and typing.

## `Waffle\Commons\Config\Config`
Implements `ConfigInterface`.
- **Typed Getters**: `getString()`, `getInt()`, `getArray()`, `getBool()`.
- **Fail-Fast**: Throws `InvalidConfigurationException` on type mismatch.

## `Waffle\Commons\Config\YamlParser`
A wrapper around the native PHP `yaml_parse_file` extension.
> [!IMPORTANT]
> The `yaml` PECL extension is required for this component to function.

## `Waffle\Commons\Config\DotEnv`
Loads `.env` files into `getenv()`/`$_ENV`.
- **Loading Order**: `.env` -> `.env.local` (override). (Logic resides in skeleton usage, component provides the loader).
