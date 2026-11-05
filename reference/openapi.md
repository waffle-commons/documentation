# OpenAPI Reference (`waffle-commons/openapi`)

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *OpenAPI 3.1 generated from `#[Route]` and typed DTOs (API-01, RFC-016)*
> **Requires:** PHP 8.5+. Depends only on `waffle-commons/contracts`, `waffle-commons/utils` (the DTO-shape reader it
> shares with the [serializer](serializer.md)) and the PSR interfaces.
> Task guide: [How to Generate an OpenAPI Document](../how-to/generate-an-openapi-document.md).

The component describes an application's HTTP API as an OpenAPI 3.1 document, built from what the code already
declares — the routes, the controller signatures and the `#[Dto]` classes — with zero hand-written YAML. Generation is
a build-time step: nothing is added to the request hot path, except the debug-only documentation route.

The routes come through `RouterInterface` and the command implements `Console\CommandInterface`, so the component
never imports a sibling's concrete class.

---

## 1. `openapi:generate` (`Command\GenerateCommand`)

```php
final readonly class GenerateCommand implements CommandInterface
{
    public const string DEFAULT_OUTPUT = 'openapi.json';
    public const string DEFAULT_TITLE = 'API';
    public const string DEFAULT_VERSION = '1.0.0';

    public function __construct(
        private RouterInterface $router,
        private OpenApiGenerator $generator = new OpenApiGenerator(),
    ) {}
}
```

```text
openapi:generate [--output=<path>|-] [--title=<title>] [--api-version=<version>]
```

| Option | Default | Meaning |
| :--- | :--- | :--- |
| `--output` | `openapi.json` | The file written, atomically (a temporary file renamed over the target); `-` writes to the standard output. |
| `--title` | `API` | The document's `info.title`. |
| `--api-version` | `1.0.0` | The document's `info.version`: your API's version, not the OpenAPI version. |

Register it in `bin/waffle` with the booted router. The skeleton does, behind `class_exists()` because the package is
one of its development dependencies:

```php
if (class_exists(GenerateCommand::class)) {
    $app->add(command: new GenerateCommand(router: $router->boot(container: $secureContainer)));
}
```

## 2. `OpenApiGenerator`

```php
final readonly class OpenApiGenerator
{
    public const string OPENAPI_VERSION = '3.1.1';
    public const array ANY_METHODS = ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'];

    public function __construct(
        private DtoShapeReader $reader = new DtoShapeReader(),
    ) {}

    public function generate(array $routes, string $title = 'API', string $version = '1.0.0'): array;
    public function encode(array $document): string;
}
```

`generate()` takes the routes `RouterInterface::getRoutes()` returns — the metadata `route:list` and `route:compile`
read — and reflects on each controller method, at build time only. The output validates against the official
OpenAPI 3.1 JSON Schema: the component's tests check it, and both template apps' documents pass.

**What each route becomes:**

| Source | Becomes |
| :--- | :--- |
| `{name}` / `{name:constraint}` in the path | a required path parameter typed from the controller parameter of the same name (`int` → `integer`, `float` → `number`, `bool` → `boolean`, else `string`); a string parameter's constraint becomes its `pattern` |
| the first parameter typed with a `#[Dto]` class | the JSON request body; required unless the parameter is nullable or defaulted |
| return type `#[Dto]` | `200` with the JSON schema of the DTO |
| return type `array` / `string` | `200` JSON / `200` HTML (no schema) |
| return type `void` | `204` |
| a nullable return type | adds `204` |
| any other or no return type | a bare `200` |

Each HTTP method of a route is an operation. Its `operationId` is the route name, suffixed with the method when the
route serves several, and numbered when two routes share a name. A route that accepts any method is documented for
the methods of `ANY_METHODS`; a method OpenAPI 3.1 cannot describe is left out, and a `HEAD` response carries no body.

**Schemas** come from `Waffle\Commons\Utils\Dto\Shape\DtoShapeReader`, the reader the serializer's normalizers use,
so the document describes exactly what the serializer accepts and emits:

- a request body is the DTO's constructor parameters — required when they have no default and are not nullable,
  with their defaults;
- a response body is its readable properties, all present (`null` included); a computed one (a `get` hook) is
  `readOnly`;
- backed enums are `enum` schemas of their backing values; `#[ListOf]` arrays are `array` schemas with `items`;
  nullable fields add `null`.

DTOs and enums are shared under `components/schemas`, named after their short class name (the full name, dotted, when
two classes share it). A DTO whose request and response shapes differ, and that the document uses on both sides, gets
two components: `Name-Input` and `Name-Output`.

## 3. Attributes (`Waffle\Commons\Contracts\OpenApi\Attribute`)

The signature cannot say everything — a summary, a tag, an error response. Three attributes add it; they live in
contracts, so they are covered by the beta8 API freeze. Without any of them the document is still valid, only terse.

```php
#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
final readonly class Operation
{
    public function __construct(
        public ?string $summary = null,
        public ?string $description = null,
        public array $tags = [],
        public bool $deprecated = false,
    ) {}
}

#[Attribute(Attribute::TARGET_METHOD | Attribute::IS_REPEATABLE)]
final readonly class Response
{
    public function __construct(
        public int $status,
        public string $description,
        public ?string $type = null,
    ) {}
}

#[Attribute(Attribute::TARGET_PARAMETER | Attribute::TARGET_PROPERTY)]
final readonly class Property
{
    /**
     * @param string|null                $description What the field holds.
     * @param string|int|float|bool|null $example     A sample value, written as the schema's `examples`.
     */
    public function __construct(
        public ?string $description = null,
        public string|int|float|bool|null $example = null,
    ) {}
}
```

| Attribute | Where | What it does |
| :--- | :--- | :--- |
| `Operation` | a method, or the class for defaults | `summary`, `description` (CommonMark), `tags`, `deprecated`. The method's summary and description win; its tags replace the class's when it has any; either level can mark an operation deprecated. |
| `Response` (repeatable) | a method | `status`, `description` and an optional `#[Dto]` body `type`. A status the signature already implies takes the new description (and body, given a type); any other status is added. |
| `Property` | a constructor parameter or a property | `description` and `example`. The request side reads the parameter, the response side the property; a promoted parameter carries it on both. |

A `Response` whose status is not between 100 and 599, or whose `type` is not a `#[Dto]` class, fails the generation
with `Exception\InvalidAttributeException`, which names the method. The component's `AttributeReader` reads all three.

```php
use Waffle\Commons\Contracts\OpenApi\Attribute as OA;

#[OA\Operation(tags: ['users'])]
final class UserController
{
    #[Route('/users/{id}')]
    #[OA\Operation(summary: 'Fetch a user', description: 'CommonMark allowed.')]
    #[OA\Response(200, 'The user.')]
    #[OA\Response(404, 'No user has this id.')]
    public function show(int $id): UserView { /* … */ }
}
```

## 4. The development documentation route (`Middleware\ApiDocsMiddleware`)

```php
final readonly class ApiDocsMiddleware implements MiddlewareInterface
{
    public const string PATH = '/_docs';
    public const string DOCUMENT_PATH = '/_docs/openapi.json';
    public const string SWAGGER_UI_VERSION = '5.33.3';

    public function __construct(
        private RouterInterface $router,
        private ResponseFactoryInterface $responseFactory,
        private StreamFactoryInterface $streamFactory,
        private string $title = 'API',
        private string $version = '1.0.0',
    ) {}
}
```

`GET /_docs` serves a Swagger UI page and `GET /_docs/openapi.json` the document generated from the routes at that
moment; every other request passes through. Mount it **in debug mode only, before the routing middleware**:
production refuses debug mode, so there both paths fall through to the router — a 404 unless one of the
application's routes matches them. The skeleton's `App\Factory\ApiDocsFactory` does exactly that, and adds nothing
when the package is not installed. The document describes every route, including those a voter protects, which is
one more reason it never runs in production.

The page loads Swagger UI 5.33.3 from jsDelivr, both files pinned by their Subresource Integrity hashes, and sends its
own `Content-Security-Policy`: scripts from jsDelivr plus the inline start-up script (by its hash), styles from
jsDelivr plus the inline styles Swagger UI writes, images from the page and `data:`, requests to the page's own origin
only. Swagger UI's online validator is off. Bumping Swagger UI changes the SRI hashes.

## 5. Worker safety

Generation keeps no state: the generator, the schema catalog, the command and the middleware are `final readonly`,
and `wfl igor` reports 0 KO.
