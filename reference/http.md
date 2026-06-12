# HTTP Reference (`waffle-commons/http`)

> **Release:** `0.1.0-beta4` &nbsp;|&nbsp; New: `UploadedFile::moveTo()` path-traversal guard (SEC-05); opt-in stream tracing (DIAG-03)
> **PSR Compliance:** PSR-7 (HTTP Messages), PSR-17 (HTTP Factories)

Strict, immutable PSR-7/17 implementation tuned for FrankenPHP worker mode. No singletons, no superglobal touching outside the explicit `GlobalsFactory`. The `ResponseEmitter` reads response bodies in 8 KiB chunks to keep large payload streaming memory-bounded.

## Message classes (PSR-7)

| Class | Implements |
| :--- | :--- |
| `Waffle\Commons\Http\Request` | `Psr\Http\Message\RequestInterface` |
| `Waffle\Commons\Http\ServerRequest` | `Psr\Http\Message\ServerRequestInterface` |
| `Waffle\Commons\Http\Response` | `Psr\Http\Message\ResponseInterface` |
| `Waffle\Commons\Http\Stream` | `Psr\Http\Message\StreamInterface` |
| `Waffle\Commons\Http\Uri` | `Psr\Http\Message\UriInterface` |
| `Waffle\Commons\Http\UploadedFile` | `Psr\Http\Message\UploadedFileInterface` |
| `Waffle\Commons\Http\Abstract\AbstractMessage` | Shared base for `Request` / `ServerRequest` / `Response`. |

## Factories (PSR-17)

| Factory | Method |
| :--- | :--- |
| `Waffle\Commons\Http\Factory\RequestFactory` | `createRequest()` |
| `Waffle\Commons\Http\Factory\ServerRequestFactory` | `createServerRequest()` |
| `Waffle\Commons\Http\Factory\ResponseFactory` | `createResponse()` |
| `Waffle\Commons\Http\Factory\StreamFactory` | `createStream()`, `createStreamFromFile()`, `createStreamFromResource()` |
| `Waffle\Commons\Http\Factory\UriFactory` | `createUri()` |
| `Waffle\Commons\Http\Factory\UploadedFileFactory` | `createUploadedFile()` |

## Secure file uploads (SEC-05)

`UploadedFile::moveTo($target)` screens its destination through
[`Assert::safePath()`](utils.md) before touching the filesystem: a target containing a
directory-traversal segment (`../`, `..\`) or a null byte is rejected, so a crafted
upload cannot escape its intended directory.

**Never build that target from raw client metadata.** `getClientFilename()` and
`getClientMediaType()` are attacker-controlled — treat them as display labels only.
Derive the stored name yourself (e.g. a generated id) and, when you must keep a
caller-supplied fragment, confine it with [`Assert::within($baseDir, $fragment)`](utils.md):

```php
$target = Assert::within($uploadDir, bin2hex(random_bytes(16)) . '.bin');
$uploadedFile->moveTo($target);
```

## `GlobalsFactory` — building a PSR-7 ServerRequest

```php
namespace Waffle\Commons\Http\Factory;

class GlobalsFactory
{
    /**
     * @param (callable(): StreamInterface)|null $bodyStreamFactory
     */
    public function __construct(?callable $bodyStreamFactory = null);

    public function createFromGlobals(): ServerRequestInterface;
}
```

`createFromGlobals()` reads `$_SERVER`, `$_GET`, `$_POST`, `$_COOKIE`, `$_FILES`, and `php://input` (the body factory is injectable for tests). The result is a fully populated `Waffle\Commons\Http\ServerRequest`.

> **Security note.** `GlobalsFactory` does **not** enforce trusted hosts. Host-header anti-poisoning is the job of `Waffle\Commons\Pipeline\Middleware\TrustedHostMiddleware`, which runs between `ErrorHandlerMiddleware` and `CoreRoutingMiddleware`.

## `ResponseEmitter` — emitting a PSR-7 Response

```php
namespace Waffle\Commons\Http\Emitter;

class ResponseEmitter implements ResponseEmitterInterface
{
    public function emit(ResponseInterface $response): void; // throws RuntimeException if headers sent
}
```

Behaviour:

- Throws `\RuntimeException` if `headers_sent()` is true.
- Sends the HTTP status line via `header()` with `replace: true`.
- For each header, sends one `header()` per value (combining for everything except `Set-Cookie`).
- Reads `$response->getBody()` in 8 KiB chunks (`while (!$body->eof()) echo $body->read(8192);`).

## Returning responses from controllers

The recommended path inside a controller is to use the helper on `Waffle\Core\BaseController`:

```php
return $this->jsonResponse(data: ['status' => 'ok']);
```

When you need finer control, use the PSR-17 factory directly:

```php
use Psr\Http\Message\ResponseFactoryInterface;

public function show(ResponseFactoryInterface $factory): ResponseInterface
{
    $response = $factory->createResponse(200)->withHeader('Content-Type', 'text/plain');
    $response->getBody()->write('Hello World');
    return $response;
}
```

## Integration with `WaffleRuntime`

The runtime wires these classes together:

```php
$runtime = new WaffleRuntime(
    globalsFactory: new GlobalsFactory(),       // optional, defaults shown
    emitter: new ResponseEmitter(),             // optional, defaults shown
);
$runtime->loop($kernel, maxRequests: 500);
```

The runtime builds a fresh `ServerRequestInterface` from `GlobalsFactory` on every iteration of its worker loop, calls `$kernel->handle($request)`, and emits via `ResponseEmitter::emit()`. See [runtime.md](runtime.md) for the loop semantics.
