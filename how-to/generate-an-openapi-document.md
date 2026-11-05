# How to Generate an OpenAPI Document

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *OpenAPI 3.1 from your routes and DTOs, and Swagger UI in development (API-01, RFC-016)*

This guide produces an OpenAPI 3.1 description of your API from the code it already has, refines it where a
signature cannot say enough, and browses it while you develop. Every class and option named here is in the
[OpenAPI reference](../reference/openapi.md).

**You need:** `waffle-commons/openapi`. The skeleton has it as a development dependency
(`composer require --dev waffle-commons/openapi`): the command and the documentation page exist only where dev
dependencies are installed. Install it as a regular dependency if your production build generates the document.

---

## 1. Register the command

In `bin/waffle`, next to the other commands, with the booted router — the skeleton's wiring:

```php
use Waffle\Commons\OpenApi\Command\GenerateCommand;

if (class_exists(GenerateCommand::class)) {
    $app->add(command: new GenerateCommand(router: $router->boot(container: $secureContainer)));
}
```

## 2. Generate the document

```bash
php bin/waffle openapi:generate --output=public/openapi.json --title="Orders API" --api-version=1.4.0
php bin/waffle openapi:generate --output=-     # to the standard output
```

The document lists every route and method, with its path parameters, request body and responses taken from the
controller signatures ([what each route becomes](../reference/openapi.md#2-openapigenerator)). Request and response
schemas come from your `#[Dto]` classes, read exactly as the serializer reads them, so the description matches what
the API accepts and returns ([How to Serialize DTOs](serialize-dtos.md)). Typed signatures pay off here: a controller
that returns `OrderView` documents its response schema; one that returns `array` documents a bare JSON body.

## 3. Say what a signature cannot

Add the contracts' attributes where a summary, a tag or an error response is missing:

```php
use Waffle\Commons\Contracts\OpenApi\Attribute as OA;

#[OA\Operation(tags: ['orders'])]                        // on the class: the default for every route
final class OrderController
{
    #[Route(path: '/orders/{id}', name: 'order_show')]
    #[OA\Operation(summary: 'Fetch an order')]
    #[OA\Response(404, 'No order has this id.')]          // a status the signature does not imply: added
    public function show(int $id): OrderView { /* … */ }
}

#[Dto]
final readonly class OrderView
{
    public function __construct(
        #[OA\Property(description: 'Public identifier.', example: 42)]
        public int $id,
    ) {}
}
```

A `Response` with a status outside 100–599, or whose `type` is not a `#[Dto]` class, stops the generation with an
`InvalidAttributeException` naming the method.

## 4. Browse it while you develop

`ApiDocsMiddleware` serves Swagger UI at `/_docs` and the live document at `/_docs/openapi.json`. Add it in debug mode
only, just before the routing middleware — the skeleton's `ApiDocsFactory::register()` does it for you:

```php
use Waffle\Commons\OpenApi\Middleware\ApiDocsMiddleware;

if ($debug && class_exists(ApiDocsMiddleware::class)) {
    $stack->add(new ApiDocsMiddleware($router, $responseFactory, $streamFactory, 'Orders API', '1.4.0'));
}
```

Production refuses debug mode, so there `/_docs` reaches the router and answers 404. Never mount it outside debug: the
document lists every route, including those a voter protects.

## 5. Keep it current

The document is a build artefact. Either generate it in the build (the API description then always matches the
image), or commit it and let CI regenerate it and fail on a difference:

```bash
php bin/waffle openapi:generate --output=public/openapi.json
git diff --exit-code public/openapi.json
```
