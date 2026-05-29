# Utils Reference (`waffle-commons/utils`)

> **Release:** `v0.1.0-beta2` &nbsp;|&nbsp; *No behavioural changes since Beta-1*

Pure-function helper services used across the ecosystem. No I/O. No state across calls. The package is the ecosystem's lightweight standard library — small enough that every consumer can require it without bloat.

## Surface

| Class | Role |
| :--- | :--- |
| `Waffle\Commons\Utils\Service\ClassParser` | Resolves the fully-qualified class/interface/trait/enum name from a PHP file using the native tokenizer. |
| `Waffle\Commons\Utils\Service\AttributeReader` | Wraps PHP 8 `ReflectionClass::getAttributes()` with type-safe convenience helpers. |
| `Waffle\Commons\Utils\Service\ReflectionInspector` | Helper for parameter / return-type / promoted-property inspection. |

All three are `final readonly`. None of them touch superglobals or filesystem state beyond what's passed to them explicitly.

## `ClassParser`

```php
namespace Waffle\Commons\Utils\Service;

final readonly class ClassParser
{
    /**
     * @return string FQCN, or '' when no class/interface/trait/enum is found.
     */
    public function className(string $path): string;
}
```

Reads `$path` with `file_get_contents`, tokenises with `token_get_all`, and walks the token stream to collect:

- the namespace declaration (handles both `namespace X;` and `namespace X { … }` bracketed forms);
- the first `class` / `interface` / `trait` / `enum` declaration's name.

Returns `''` when:

- the file does not exist or is not readable;
- the contents cannot be loaded;
- no top-level class-like declaration is present.

### Why token-based, not regex

Beta-1 replaced an older regex-based scanner because:

- PHP 8.x features (`readonly` classes, enums, bracketed namespaces) confuse naive regex matchers;
- regex scanners have ReDoS surface;
- `token_get_all` is the official PHP API for this use case and handles every grammar quirk.

This is the rationale captured in the class's docblock — "audit: 'Eradicate ReflectionTrait'".

## `AttributeReader`

A thin wrapper that returns typed iterables for `#[Attribute]` discovery on classes and methods. The typical usage is in middleware that wants to inspect a controller method's attributes (e.g. `CsrfMiddleware` looking up `#[RequiresCsrfToken]`, `SecureContainer::analyze()` looking up `#[Voter]` / `#[PublicAccess]`).

```php
$attrs = $reader->onMethod(MyController::class, 'index', RequiresCsrfToken::class);
// list<RequiresCsrfToken>
```

The reader internally uses `ReflectionClass::getAttributes(Attribute::class, ReflectionAttribute::IS_INSTANCEOF)` and calls `->newInstance()`, returning a list of attribute instances ready to use.

## `ReflectionInspector`

Helpers for parameter introspection — used by the framework's `ControllerArgumentResolver` (in `waffle-commons/waffle`) and various dependency-injection strategies.

Typical operations:

- "Is parameter `$p` a `\ReflectionParameter` whose declared type is a DTO with promoted properties?"
- "Is the return type of `$method` a `ResponseInterface`?"
- "Does the parameter carry a specific attribute?"

All operations are stateless reflections — they don't cache. Callers that need caching layer it themselves.

## Worker-mode safety

All three classes are `final readonly` with no constructor parameters and no instance state. Safe to share across worker requests — and in fact intended to be reused.

## What's deliberately *not* here

- **HTTP helpers** — those live in `waffle-commons/http`.
- **String / array helpers** — PHP's standard library plus a recent runtime makes them unnecessary.
- **Container interaction** — DI is `waffle-commons/container`'s job.
- **Logging** — `waffle-commons/log`.

The package is small on purpose. New helpers are added only when ≥2 components would consume them.

## Related

- [core.md](core.md) — `ControllerArgumentResolver` consumes `ReflectionInspector`.
- [event-dispatcher.md](event-dispatcher.md) — `ClassParser` is used by `AppKernelFactory::discoverAndRegisterListeners()`.
