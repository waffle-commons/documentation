# Serializer Reference (`waffle-commons/serializer`)

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *Compiled DTO serialization and JSON content negotiation (API-02, RFC-016)*
> **Requires:** PHP 8.5+, `psr/http-server-middleware`, `psr/log`. Depends only on `waffle-commons/contracts`,
> `waffle-commons/utils` (the DTO-shape reader it shares with the [OpenAPI generator](openapi.md)) and the PSR interfaces.
> Task guide: [How to Serialize DTOs](../how-to/serialize-dtos.md).

The serializer turns `#[Dto]` objects into JSON and JSON request bodies into `#[Dto]` objects, at the HTTP boundary.
It is **not** a general-purpose serializer: it handles exactly the DTO shapes that `utils`' `DtoShapeReader` reads —
the same shapes the OpenAPI document describes, so the specification and the wire format agree. `data`'s hydrator
stays inside the persistence layer.

---

## 1. The contract (`Waffle\Commons\Contracts\Serializer`)

```php
interface DtoSerializerInterface
{
    public function supports(string $class): bool;

    /** @template T of object  @param class-string<T> $class  @return T */
    public function denormalize(array $data, string $class): object;

    /** @template T of object  @param class-string<T> $class  @return T */
    public function deserialize(string $json, string $class): object;

    public function serialize(object $dto): string;
}

interface SerializerExceptionInterface extends Throwable {}
```

The framework and the other components use the serializer through this interface, without importing it: the
controller argument resolver hydrates request bodies with it, the response converter writes returned DTOs with it,
and the queue's envelope codec encodes messages with it. A DTO is always built through its constructor, so its `set`
hooks validate every value. Data that does not fit the DTO is a validation failure that names the field. Every
`SerializerExceptionInterface` carries the HTTP status the error handler renders as its exception code.

**How the kernel picks it up.** When the container holds a `DtoSerializerInterface`, `AbstractKernel` hands it to the
controller dispatcher: `#[Dto]` parameters are hydrated by the serializer and returned DTOs are written by it.
Without one, the kernel's built-in reflection hydration serves, as before beta7.

## 2. `#[ListOf]` (`Waffle\Commons\Contracts\Attribute`)

```php
#[Attribute(Attribute::TARGET_PARAMETER | Attribute::TARGET_PROPERTY)]
final readonly class ListOf
{
    /** @param 'int'|'float'|'string'|'bool'|class-string $type The element type. */
    public function __construct(
        public string $type,
    ) {}
}
```

PHP has no generics: a member declared `array` says nothing about what it holds. `#[ListOf]` declares the element
type — a builtin scalar name, or the class-string of a `#[Dto]` or of a backed enum. The shape reader reads it, the
serializer checks and (de)normalizes every element against it, and the OpenAPI generator emits it as the schema's
`items`. On a promoted constructor parameter it applies to both the parameter and the property. It describes one
level: a list of lists is not supported. Keep the `list<…>` docblock too — Mago reads the docblock, the reader
reads the attribute:

```php
#[Dto]
final readonly class OrderInput
{
    /**
     * @param list<OrderLine> $lines
     * @param list<string>    $tags
     */
    public function __construct(
        #[ListOf(OrderLine::class)] public array $lines,
        #[ListOf('string')] public array $tags = [],
    ) {}
}
```

## 3. `Serializer`

```php
final readonly class Serializer implements DtoSerializerInterface
{
    public function __construct(
        private NormalizerProviderInterface $normalizers,
        private JsonCodec $codec = new JsonCodec(),
    ) {}

    public function supports(string $class): bool;
    public function normalize(object $dto): array;
    public function denormalize(array $data, string $class): object;
    public function serialize(object $dto): string;
    public function deserialize(string $json, string $class): object;
}
```

The `new` expressions in parameter defaults are PHP 8.1+ initializers; the class itself is `final readonly`, so every
promoted property is immutable.

**Mapping rules:**

- **In:** the constructor parameters. Hydration always goes through the constructor, with named arguments, so `set`
  hooks validate every value; nothing writes a property directly.
- **Out:** the readable properties — `get` hooks run, write-only virtual properties are skipped. A `password`
  parameter can come in while a `passwordHash` property goes out.
- **Types:** `int`, `float` (an integer is accepted), `string`, `bool`, backed enums (as their backing value), nested
  `#[Dto]`s and `#[ListOf]` lists, each optionally nullable.
- **Keys** are the property and parameter names, unchanged. An unknown key is ignored. An absent key takes the
  parameter's default, `null` for a nullable parameter without one, and is an error otherwise.
- `null` values are written out; a DTO without readable properties is `{}`; floats keep their fraction (`2.0`).

## 4. `Json\JsonCodec`

```php
final readonly class JsonCodec
{
    public const int DEFAULT_MAX_BYTES = 1_048_576;
    public const int DEFAULT_MAX_DEPTH = 64;

    public function __construct(
        public private(set) int $maxBytes = self::DEFAULT_MAX_BYTES,
        public private(set) int $maxDepth = self::DEFAULT_MAX_DEPTH,
    ) {}

    public function decodeObject(string $json): array;
    public function encode(array|stdClass $value): string;
}
```

A body longer than `$maxBytes` is refused before it is parsed (413); one nested deeper than `$maxDepth`, or that is
not a JSON object, is malformed (400).

## 5. Normalizers: compiled and interpreted

```php
interface NormalizerProviderInterface
{
    public function get(string $class): ?DtoNormalizerInterface;
}

interface DtoNormalizerInterface
{
    /** @return array<string, int|float|string|bool|array|stdClass|null> */
    public function normalize(object $dto): array;

    /** @param string $path Where `$data` sits in the payload, for error messages. */
    public function denormalize(array $data, string $path = ''): object;
}
```

These two interfaces are the component's own seam between its runtime and the code it generates (they are not in
contracts: no other component consumes them).

| Provider | When | How |
| :--- | :--- | :--- |
| The generated index (`Compiler\NormalizerCompiler`) | production | One generated `final readonly` class per DTO: plain property reads out, a plain constructor call in. No reflection at request time. |
| `Normalizer\ShapeNormalizerProvider` | development | Reads the DTO shape on demand and normalizes by reflection. No build step. |

Both run every check through `Normalizer\Payload`, so they accept, reject and word errors identically; the test suite
requires the compiled output to match the interpreted one byte for byte.

```php
final readonly class ShapeNormalizerProvider implements NormalizerProviderInterface
{
    public function __construct(
        private DtoShapeReader $reader = new DtoShapeReader(),
    ) {}
}

final readonly class FallbackNormalizerProvider implements NormalizerProviderInterface
{
    public function __construct(
        private NormalizerProviderInterface $primary,
        private NormalizerProviderInterface $fallback,
    ) {}
}

final readonly class CompiledNormalizerLoader
{
    public const string ENV_FLAG = 'WAFFLE_AOT';
    public const string DEFAULT_DIRECTORY = 'var/cache/serializer';
    public const string REBUILD_COMMAND = 'bin/waffle serializer:compile';

    public function __construct(
        private string $directory,
        private string $namespace = NormalizerCompiler::DEFAULT_NAMESPACE,
        private string $index = NormalizerCompiler::DEFAULT_INDEX,
        private LoggerInterface $logger = new NullLogger(),
        private NormalizerProviderInterface $interpreted = new ShapeNormalizerProvider(),
    ) {}

    public function load(): NormalizerProviderInterface;
}
```

`CompiledNormalizerLoader::load()` returns the compiled index when `WAFFLE_AOT=1` and the generated files load, backed
by the interpreted provider for any DTO the build did not cover (`FallbackNormalizerProvider`). Otherwise — the flag
off, the files missing or broken — it returns the interpreted provider, with a log line saying why. Every successful
load also logs that the build is not checked against the current code: re-run `serializer:compile` after any change to
a DTO, as for `container:compile` ([AOT reference](aot.md)).

```php
final readonly class NormalizerCompiler
{
    public const string DEFAULT_NAMESPACE = 'Waffle\\Generated\\Serializer';
    public const string DEFAULT_INDEX = 'CompiledNormalizers';

    public function __construct(
        private DtoShapeReader $reader = new DtoShapeReader(),
    ) {}

    /** @return array<string, string> The generated files, one class each: file name => PHP source, sorted by name. */
    public function compile(
        array $classes,
        string $namespace = self::DEFAULT_NAMESPACE,
        string $index = self::DEFAULT_INDEX,
    ): array;
}
```

The generated code is deterministic and passes Mago's linter and analyzer like hand-written code.

## 6. `serializer:compile` (`Command\CompileCommand`)

```text
serializer:compile [--source=<dir>] [--output=<dir>] [--namespace=<namespace>]
```

| Option | Default | Meaning |
| :--- | :--- | :--- |
| `--source` | `src` (`CompileCommand::DEFAULT_SOURCE`) | The directory scanned for `#[Dto]` classes. |
| `--output` | `var/cache/serializer` (`CompiledNormalizerLoader::DEFAULT_DIRECTORY`) | Where the normalizers are written. |
| `--namespace` | `Waffle\Generated\Serializer` | The namespace of the generated classes. |

It writes one file per DTO class, each atomically. A file a previous run generated for a DTO that no longer exists is
removed; any other file in the directory is left alone. Register it in `bin/waffle` next to the console commands:
`$app->add(command: new CompileCommand());`.

## 7. `Middleware\ContentNegotiationMiddleware`

```php
final readonly class ContentNegotiationMiddleware implements MiddlewareInterface
{
    public const string JSON = 'application/json';

    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface;
    public static function acceptsJson(string $accept): bool;
}
```

It settles the one question a JSON-only API has: does the client accept `application/json`? It reads `Accept` as
RFC 9110 defines it — the most specific matching range decides (`application/json`, then `application/*`, then
`*/*`), and a weight of 0 excludes — and refuses a request that rules JSON out with a 406
(`NotAcceptableException`) before any controller runs. No `Accept` header accepts anything. Every response that
passes varies on `Accept`. Mount it inside the error-handler middleware, and only in front of JSON routes: it refuses
a request that asks for another type, such as a `text/event-stream` subscription or an HTML page. Other
representations are post-v1.

## 8. Errors

| Exception (`Waffle\Commons\Serializer\Exception`) | Status | When |
| :--- | :--- | :--- |
| `InvalidPayloadException` | 422 | A field is missing, mistyped, not one of an enum's values, or rejected by a `set` hook. `getField()` is its path from the root: `email`, `shipping.city`, `lines[2].quantity`. |
| `MalformedPayloadException` | 400 | The body is not JSON, or nests deeper than the codec's limit. |
| `PayloadTooLargeException` | 413 | The body is longer than the codec's limit; it is not parsed. |
| `NormalizerException` | 500 | A class has no normalizer, or is a `#[Dto]` the serializer cannot represent: a wiring mistake. |
| `NotAcceptableException` | 406 | The request's `Accept` header rules JSON out. |

The error handler renders them; a 422 shows the message and the field, as for any validation failure.

## 9. Worker safety

`Serializer`, `JsonCodec`, the providers, the loader, the compiler, the command and the middleware are all
`final readonly` and keep nothing between calls; `wfl igor` reports 0 KO. The compiled index is loaded once, at boot.
