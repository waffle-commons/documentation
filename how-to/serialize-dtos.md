# How to Serialize DTOs

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *JSON ↔ `#[Dto]` at the HTTP boundary, compiled for production (API-02, RFC-016)*

This guide hands JSON request bodies and responses to `waffle-commons/serializer`: registered once, it hydrates every
`#[Dto]` controller parameter and writes every `#[Dto]` a controller returns, reflection-free in production. Every
class named here is in the [serializer reference](../reference/serializer.md).

**You need:** `composer require waffle-commons/serializer`.

---

## 1. Register the serializer

Build it in your kernel factory and put it on the container under the contract. The kernel uses it as soon as the
container holds a `DtoSerializerInterface`; without one, its built-in reflection hydration serves. This is the
workspace's factory method:

```php
use Waffle\Commons\Contracts\Serializer\DtoSerializerInterface;
use Waffle\Commons\Serializer\Normalizer\CompiledNormalizerLoader;
use Waffle\Commons\Serializer\Serializer;

public static function buildSerializer(string $root, LoggerInterface $logger): Serializer
{
    return new Serializer(
        new CompiledNormalizerLoader(
            directory: $root . '/' . CompiledNormalizerLoader::DEFAULT_DIRECTORY,
            logger: $logger,
        )->load(),
    );
}

// where the other services are set:
$container->set(DtoSerializerInterface::class, self::buildSerializer($root, $logger));
```

`new CompiledNormalizerLoader(…)->load()` uses PHP 8.4's `new` without wrapping parentheses. The loader returns the
compiled normalizers when `WAFFLE_AOT=1` and they exist, the interpreted ones otherwise (step 4). Reuse the same
serializer anywhere else that needs one, such as the queue's envelope codec.

## 2. Write the DTOs

**Input** — what a request body must contain. Validation lives in PHP 8.5 `set` hooks, and since a hooked property
cannot be `readonly`, the class is `final class` with `public private(set)` properties:

```php
use Waffle\Commons\Contracts\Attribute\Dto;
use Waffle\Commons\Contracts\Attribute\ListOf;
use Waffle\Exception\ValidationException; // waffle-commons/waffle, as in the skeleton's HelloInput

#[Dto]
final class OrderInput
{
    /**
     * @param list<OrderLine> $lines
     */
    public function __construct(
        public private(set) string $email {
            set(string $value) {
                if (filter_var($value, FILTER_VALIDATE_EMAIL) === false) {
                    throw new ValidationException(message: 'Field "email" must be an email address.', field: 'email');
                }
                $this->email = $value;
            }
        },
        #[ListOf(OrderLine::class)]
        public private(set) array $lines,
        public private(set) ?string $note = null,
    ) {}
}
```

**Output** — what a response contains: its readable properties. Without hooks it can be `final readonly`:

```php
#[Dto]
final readonly class OrderView
{
    public function __construct(
        public int $id,
        public OrderStatus $status, // a backed enum, written as its value
    ) {}
}
```

The types the serializer handles: `int`, `float`, `string`, `bool`, backed enums, nested `#[Dto]`s and `#[ListOf]`
lists of any of them, each optionally nullable. A list of lists is not supported. Keep the `list<…>` docblock next to
`#[ListOf]`: Mago reads the docblock, the serializer reads the attribute.

## 3. Use them in controllers

```php
#[Route(path: '/orders', name: 'order_create', methods: ['POST'])]
public function create(OrderInput $input): OrderView
{
    return $this->orders->place($input);
}
```

- The `#[Dto]` parameter is built from the JSON body, through its constructor: every hook runs.
- The returned DTO is written as the JSON response body.
- A field that is missing, mistyped or refused by a hook answers **422** with its path (`email`,
  `lines[2].quantity`); a body that is not JSON answers 400, one over 1 MiB answers 413.
- Unknown keys are ignored; an absent key takes the parameter's default, `null` if it is nullable, and is an error
  otherwise.

The OpenAPI generator reads the same shapes, so `openapi:generate` documents exactly this contract
([How to Generate an OpenAPI Document](generate-an-openapi-document.md)).

## 4. Compile the normalizers for production

In development the serializer reflects on each DTO when it first meets it. In production, compile one normalizer per
DTO at build time and load them on the `WAFFLE_AOT=1` fast path. Register the command in `bin/waffle`:

```php
use Waffle\Commons\Serializer\Command\CompileCommand;

$app->add(command: new CompileCommand());
```

Then, in the image build, after the application code is copied:

```bash
php bin/waffle serializer:compile --source=src   # the workspace keeps its code in app/: --source=app
```

and run the application with `WAFFLE_AOT=1`. The files land in `var/cache/serializer`. A DTO the build missed still
works, through the interpreted fallback, and a missing or broken build falls back entirely, with a log line saying why.
The build is not checked against the code at run time: **re-run the command after any change to a DTO**, as for
`container:compile` ([How to Enable AOT](enable-aot.md)).

## 5. Refuse clients that do not accept JSON (optional)

`ContentNegotiationMiddleware` answers 406 to a request whose `Accept` header rules JSON out, before any controller
runs, and adds `Vary: Accept` to the others:

```php
use Waffle\Commons\Serializer\Middleware\ContentNegotiationMiddleware;

$stack->add(middleware: new ContentNegotiationMiddleware()); // e.g. just before the routing middleware
```

Mount it after the error handler (which renders the 406) and **after** the metrics and probe middlewares: a Prometheus
scrape asks for a text format and would be refused. Mount it only if every route behind it answers JSON — a
`text/event-stream` subscription or an HTML page would be refused too.

## 6. Test it

Send JSON through the real pipeline with `waffle-commons/testing` ([How to Test Through the Kernel](test-through-the-kernel.md)):

```php
$response = $this->createClient()->jsonRequest('POST', '/orders', [
    'email' => 'not-an-address',
    'lines' => [],
]);

self::assertStatus(422, $response); // the access rule runs first: a denied request answers 403
```

The security layer runs before the arguments are hydrated, so give the route its access rule (`#[Voter]` or
`#[PublicAccess]`) as for any other ([How to Secure a Controller](secure-a-controller.md)).
