# Utils Reference (`waffle-commons/utils`)

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; New: the DTO shape reader (`DtoShapeReader`) & a PSR-20 clock (`SystemClock`)

Pure-function helper services used across the ecosystem. No I/O. No state across calls. The package is the ecosystem's lightweight standard library — small enough that every consumer can require it without bloat.

## Surface

| Class | Role |
| :--- | :--- |
| `Waffle\Commons\Utils\Service\ClassParser` | Resolves the fully-qualified class/interface/trait/enum name from a PHP file using the native tokenizer. |
| `Waffle\Commons\Utils\Service\AttributeReader` | Wraps PHP 8 `ReflectionClass::getAttributes()` with type-safe convenience helpers. |
| `Waffle\Commons\Utils\Service\ReflectionInspector` | Helper for parameter / return-type / promoted-property inspection. |
| `Waffle\Commons\Utils\Assert` | Static entry point for input validation **and** cleansing — drops straight into PHP 8.5 property-hook setters. |
| `Waffle\Commons\Utils\Validation\AssertValidator` | Injectable `ValidatorInterface` (DX-05) wrapping the static `Assert` facade: runs a `SelfValidatingInterface` object's `assertValid()` and converts the first thrown violation into a `ValidationResult` (with `Violation` entries). Mockable in tests. |
| `Waffle\Commons\Utils\Dto\Shape\DtoShapeReader` | Reads the shape of a `#[Dto]` class at build time — the keys it accepts and emits — for the serializer and the OpenAPI generator (*since `0.1.0-beta7`*). See [below](#dtoshapereader--the-shape-of-a-dto). |
| `Waffle\Commons\Utils\Clock\SystemClock` | PSR-20 `ClockInterface` on the system time, in a fixed time zone (*since `0.1.0-beta7`*). See [below](#systemclock--psr-20-clock). |

The three `Service\*` classes are `final readonly`. None of them touch superglobals or filesystem state beyond what's passed to them explicitly. `Assert` has its own section below.

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

## `Assert` — input validation & cleansing

`Assert` validates **and returns the cleansed value** in a single call, so it drops straight into a PHP 8.5 property-hook short-setter — the value is checked and normalised in one line:

```php
use Waffle\Commons\Utils\Assert;

#[Dto]
final class RegistrationInput
{
    // validates the address, then stores it trimmed + lower-cased
    public private(set) string $email {
        set => Assert::email($value);
    }
}
```

Every method is `public static`, pure and deterministic. The class is `final readonly` with a private constructor: call the methods statically, never instantiate it.

It is **complementary** to the collecting `Contracts\Validation\ValidatorInterface` (which gathers *every* violation). `Assert` is the *fail-fast* cleanser that throws on the first bad value — exactly what a property hook wants.

### Composition (Strategy A — vertical traits)

The surface is split into four concern families, each a trait composed into `Assert`. You always call through the one entry point (`Assert::email(...)`), but each family stays small and focused:

| Family (trait) | Methods |
| :--- | :--- |
| `StringAssertionsTrait` | `email`, `uuid`, `length`, `regex`, `notEmpty` |
| `NumericAssertionsTrait` | `greaterThanOrEqual`, `lessThanOrEqual`, `range`, `positive` |
| `NetworkAssertionsTrait` | `ip`, `port`, `cidr`, `isPublicIp`, `ipInCidr` |
| `FileAssertionsTrait` | `exists`, `readable`, `writable`, `safePath`, `within` |

### Methods

Each *assertion* accepts an optional trailing `?string $message` to override the default error text. Numeric methods return the number unchanged; string methods return the **normalised** value. The two SSRF predicates (`isPublicIp`, `ipInCidr`) instead return `bool` and never throw — they are matchers, not assertions.

| Method | Validates | Returns |
| :--- | :--- | :--- |
| `email(string, ?string): string` | RFC e-mail | trimmed + lower-cased |
| `uuid(string, ?string): string` | UUID v4 / v5 | trimmed + lower-cased |
| `length(string, int $min, int $max, ?string): string` | inclusive `mb_strlen` window | value unchanged |
| `regex(string, string $pattern, ?string): string` | PCRE match | value unchanged |
| `notEmpty(string, ?string): string` | non-blank after trim | trimmed |
| `greaterThanOrEqual(int\|float, int\|float, ?string)` | `>= threshold` | value unchanged |
| `lessThanOrEqual(int\|float, int\|float, ?string)` | `<= threshold` | value unchanged |
| `range(int\|float, int\|float $min, int\|float $max, ?string)` | inclusive window | value unchanged |
| `positive(int\|float, ?string)` | `> 0` | value unchanged |
| `ip(string, ?string): string` | IPv4 / IPv6 | trimmed + lower-cased |
| `port(int, ?string): int` | `1..65535` | value unchanged |
| `cidr(string, ?string): string` | `address/prefix`, family-bounded prefix | trimmed + lower-cased |
| `isPublicIp(string): bool` | IP is publicly routable (not loopback / RFC 1918 / RFC 4193 / link-local / CGNAT / multicast / reserved) | `bool` — fail-closed: malformed ⇒ `false` |
| `ipInCidr(string, string): bool` | IP falls inside a CIDR block (family-aware; v4 never matches a v6 block) | `bool` |
| `exists(string, ?string): string` | path exists | the safe (trimmed) path |
| `readable(string, ?string): string` | path readable | the safe (trimmed) path |
| `writable(string, ?string): string` | path writable | the safe (trimmed) path |
| `safePath(string, ?string): string` | no `../` / `..\` traversal segment (plus null-byte / blank) | the safe (trimmed) path |
| `within(string $base, string, ?string): string` | resolved target stays inside the existing `$base` directory | lexically-normalised absolute path |

The file methods first screen each path for the **null-byte injection** vector (`\0`) and for emptiness before touching the filesystem. `safePath()` / `within()` additionally reject **directory traversal** (`../`, `..\`) so user-supplied path fragments cannot escape their intended location (SEC-05). `isPublicIp()` / `ipInCidr()` back the HTTP client's **SSRF** defence (SEC-02) — see [http-client.md](http-client.md).

### Failure: `ValidationException`

Every assertion throws `Waffle\Commons\Utils\Exception\ValidationException`, which **extends the SPL `\InvalidArgumentException`** *and* **implements `Contracts\Exception\Validation\ValidationExceptionInterface`**. One type satisfies both contracts:

- a plain `catch (\InvalidArgumentException $e)` works;
- the `JsonErrorRenderer` recognises the interface and emits an **RFC 7807 HTTP 422**;
- the code defaults to `422`, matching `Waffle\Exception\ValidationException`.

`Assert` checks are *value-level*, so the thrown exception's `getField()` is `null` — the assertion does not know the property name. When you need the `field` key populated in the 422 payload, throw `Waffle\Exception\ValidationException` with `field:` yourself inside a full hook. See [How-To: Validate & Cleanse Input](../how-to/validate-input.md).

### Worker-mode safety (`Assert`)

`Assert` and its four traits declare **no properties of any kind** — no instances, no `static` caches. Nothing survives a request, so it is safe under FrankenPHP resident-worker mode. This is enforced two ways: a reflection/determinism PHPUnit test, and a static `igor-php` audit (`composer igor`) that fails on any worker-unsafe state.

## `DtoShapeReader` — the shape of a `#[Dto]`

*Since `0.1.0-beta7` (RFC-016, CARRY-04).* `Waffle\Commons\Utils\Dto\Shape\DtoShapeReader` (`utils/src/Dto/Shape/DtoShapeReader.php`) reads a `#[Dto]` class at build time: the keys it accepts (its constructor parameters) and the keys it emits (its readable properties). The serializer's normalizer compiler ([serializer.md](serializer.md)) and the OpenAPI schema builder ([openapi.md](openapi.md)) both read DTOs through it, so the generated document describes exactly what the serializer accepts and emits.

```php
final readonly class DtoShapeReader
{
    /** @param class-string $class  @throws UnsupportedDtoShapeException */
    public function read(string $class): DtoShape;

    /**
     * Reads the given DTOs and every `#[Dto]` they reach, directly or through a
     * list, on either side. A cycle is followed once.
     *
     * @param class-string ...$roots
     * @return array<class-string, DtoShape> Keyed by class name, in discovery order.
     * @throws UnsupportedDtoShapeException
     */
    public function readGraph(string ...$roots): array;
}
```

- **Inputs** are the constructor parameters, in declaration order. A payload is hydrated through the constructor with named arguments, so the `set` hooks and the constructor body always run; nothing writes a property directly.
- **Outputs** are the readable properties: public, non-static, and not a virtual property with only a `set` hook. A virtual property with a `get` hook is a *computed* output. Asymmetric visibility (`public private(set)`) and `readonly` change nothing for reading.
- The two sides are matched **by name only**, on purpose: a `password` parameter hashed into a `passwordHash` property is accepted under one name and emitted under the other.
- A field is an `int`, a `float`, a `string`, a `bool`, a backed enum, a nested `#[Dto]` (`self` included) or an `array` whose element type `#[ListOf]` declares; any of them may be nullable (`?T`, `T|null`). A default value is `null`, a scalar, a backed enum case or `[]`. Anything else throws `UnsupportedDtoShapeException`, which names the member.

The reader reflects on every call and caches nothing, so it holds no state across worker requests.

### The shape value objects

Each is a `final readonly` class whose fields use asymmetric visibility (`public private(set)`, PHP 8.4): readable everywhere, written only by the constructor.

```php
namespace Waffle\Commons\Utils\Dto\Shape;

final readonly class DtoShape
{
    /**
     * @param class-string      $class
     * @param list<InputField>  $inputs
     * @param list<OutputField> $outputs
     */
    public function __construct(
        public private(set) string $class,
        public private(set) array $inputs,
        public private(set) array $outputs,
    ) {}

    public function input(string $name): ?InputField;
    public function output(string $name): ?OutputField;

    /** @return list<class-string> The #[Dto] classes referred to, directly or as list elements, each once, in first-seen order. */
    public function dependencies(): array;
}

final readonly class InputField
{
    /** Whether the payload must carry the key: no default value and not nullable. */
    public private(set) bool $required;

    /** @param int|float|string|bool|BackedEnum|array{}|null $default The default value, when `$hasDefault`. */
    public function __construct(
        public private(set) string $name,
        public private(set) FieldType $type,
        public private(set) bool $hasDefault = false,
        public private(set) int|float|string|bool|BackedEnum|array|null $default = null,
    ) {}
}

final readonly class OutputField
{
    public function __construct(
        public private(set) string $name,
        public private(set) FieldType $type,
        public private(set) bool $computed = false,
    ) {}
}
```

An absent key falls back to the parameter's default, and a nullable parameter without a default receives `null`, so only a parameter with neither is `required`. A `computed` output has no backing value: the OpenAPI schema marks it read-only.

`FieldType` has a private constructor and four named constructors, so every instance is a valid combination: a scalar carries no class, a `#[Dto]` its class-string, a backed enum its class-string and backing kind, a list its element type (never nullable, never a list).

```php
final readonly class FieldType
{
    /** @param class-string|null $class */
    private function __construct(
        public private(set) TypeKind $kind,
        public private(set) bool $nullable,
        public private(set) ?string $class = null,
        public private(set) ?TypeKind $backing = null,
        public private(set) ?FieldType $item = null,
    ) {}

    /** @throws InvalidArgumentException When the kind is not a scalar kind. */
    public static function scalar(TypeKind $kind, bool $nullable = false): self;

    /** @param class-string $class */
    public static function dto(string $class, bool $nullable = false): self;

    /** @param class-string<BackedEnum> $class  @throws InvalidArgumentException When the backing kind is neither Int nor String. */
    public static function enum(string $class, TypeKind $backing, bool $nullable = false): self;

    /** @throws InvalidArgumentException When the element type is nullable or a list itself. */
    public static function listOf(self $item, bool $nullable = false): self;

    /** @return class-string|null The #[Dto] class this type refers to, directly or as a list element. */
    public function referencedDto(): ?string;

    /** The type in a PHP-like notation, for messages: `int`, `?string`, `list<App\Dto\Line>`, `?App\Status`. */
    public function describe(): string;
}
```

```php
namespace Waffle\Commons\Utils\Dto\Shape\Enum;

enum TypeKind: string
{
    case Int = 'int';
    case Float = 'float';
    case String = 'string';
    case Bool = 'bool';
    case Enum = 'enum';
    case Dto = 'dto';
    case List = 'list';

    /** Whether the kind is one of the four JSON scalars. */
    public function isScalar(): bool;
}
```

### Failure: `UnsupportedDtoShapeException`

```php
namespace Waffle\Commons\Utils\Exception;

final class UnsupportedDtoShapeException extends LogicException
{
    /**
     * @param string $dtoClass The class being read.
     * @param string $member   The member at fault (`constructor parameter $tags`, `property $total`), or `''` for the class itself.
     * @param string $reason   Why it is unsupported, phrased to follow the member.
     */
    public function __construct(
        public private(set) string $dtoClass,
        public private(set) string $member,
        string $reason,
    ) {}
}
```

The message reads `Cannot read the DTO shape of "App\Dto\Order": constructor parameter $tags …`. It is raised at build time, while the shape is read, and reports a mistake in the DTO, never a client error: it extends `LogicException` and maps to no HTTP status.

## `SystemClock` — PSR-20 clock

*Since `0.1.0-beta7` (CARRY-04).* `Waffle\Commons\Utils\Clock\SystemClock` (`utils/src/Clock/SystemClock.php`) implements PSR-20's `Psr\Clock\ClockInterface` on the system time. Code that needs the current time takes a `ClockInterface` by injection and receives this clock in production; tests swap in a frozen one ([testing.md](testing.md)).

```php
final readonly class SystemClock implements ClockInterface
{
    public function __construct(
        private DateTimeZone $timezone = new DateTimeZone('UTC'),
    ) {}

    #[\Override]
    public function now(): DateTimeImmutable;
}
```

The time zone is fixed at construction — UTC by default, through a `new` expression in the parameter default (PHP 8.1) — instead of being read from the ambient `date.timezone` setting, so the same code yields the same instant in every environment.

## Worker-mode safety

The three `Service\*` classes are `final readonly` with no constructor parameters and no instance state; `Assert` and its traits declare no properties at all and expose only `static` methods. Either way nothing survives a request — safe to share across worker requests, and in fact intended to be reused. `DtoShapeReader` is `final readonly` with no state and reflects on every call; `SystemClock`'s only field is its readonly time zone, and every `now()` reads the time again.

## What's deliberately *not* here

- **HTTP helpers** — those live in `waffle-commons/http`.
- **Generic string / array manipulation** — PHP's standard library plus a recent runtime makes it unnecessary. (Input *validation & cleansing* is the deliberate exception — see [`Assert`](#assert--input-validation--cleansing).)
- **Container interaction** — DI is `waffle-commons/container`'s job.
- **Logging** — `waffle-commons/log`.

The package is small on purpose. New helpers are added only when ≥2 components would consume them.

## Related

- [core.md](core.md) — `ControllerArgumentResolver` consumes `ReflectionInspector`.
- [event-dispatcher.md](event-dispatcher.md) — `ClassParser` is used by `AppKernelFactory::discoverAndRegisterListeners()`.
- [serializer.md](serializer.md) and [openapi.md](openapi.md) — both read DTOs through `DtoShapeReader`.
- [testing.md](testing.md) — the frozen clock that replaces `SystemClock` in tests.
