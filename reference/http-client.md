# HTTP Client Reference (`waffle-commons/http-client`)

> **Release:** `v0.1.0-beta1`
> **PSR Compliance:** PSR-18 (`Psr\Http\Client\ClientInterface`), consumes PSR-7 messages, PSR-17 factories

A high-performance PSR-18 HTTP client tuned for FrankenPHP resident-worker proxying. Holds a single persistent `\CurlHandle` reused via `curl_reset()` across every `sendRequest()` so libcurl's DNS cache and keep-alive pool stay warm. Response bodies are streamed in 8 KiB chunks directly into a PSR-7 stream backed by `php://temp`; the full body is never materialised as a string.

## `Client` — surface

```php
namespace Waffle\Commons\HttpClient;

final readonly class Client implements ClientInterface
{
    public const int CONNECT_TIMEOUT_MS = 1_000;
    public const int TIMEOUT_MS         = 10_000;
    public const int CHUNK_SIZE         = 8_192;

    public function __construct(
        private ResponseFactoryInterface $responseFactory,
        private StreamFactoryInterface   $streamFactory,
    );

    public function sendRequest(RequestInterface $request): ResponseInterface;
}
```

## Beta-1 / SEC-03 — SSRF protocol allowlist

`Client::applyRequest()` enforces:

```php
CURLOPT_PROTOCOLS       => CURLPROTO_HTTP | CURLPROTO_HTTPS,
CURLOPT_REDIR_PROTOCOLS => CURLPROTO_HTTP | CURLPROTO_HTTPS,
```

This blocks SSRF pivots via `file://`, `gopher://`, `dict://`, `ldap://`, `telnet://`, … even when a caller-supplied URL or a server-supplied `Location` header tries to switch protocols mid-flight.

## Hardcoded cURL defaults

The client enforces a security baseline that callers cannot lower:

| Option | Value | Why |
| :--- | :--- | :--- |
| `CURLOPT_PROTOCOLS` | `CURLPROTO_HTTP \| CURLPROTO_HTTPS` | SSRF allowlist (request URL). |
| `CURLOPT_REDIR_PROTOCOLS` | `CURLPROTO_HTTP \| CURLPROTO_HTTPS` | SSRF allowlist (redirect target). |
| `CURLOPT_SSL_VERIFYPEER` | `true` | Forces certificate validation. |
| `CURLOPT_SSL_VERIFYHOST` | `2` | Forces hostname match. |
| `CURLOPT_FOLLOWLOCATION` | `false` | The client never silently follows redirects. |
| `CURLOPT_CONNECTTIMEOUT_MS` | `1_000` | 1-second connect ceiling. Cannot be raised. |
| `CURLOPT_TIMEOUT_MS` | `10_000` | 10-second total ceiling. Cannot be raised. A hung legacy backend must never lock a worker thread. |
| `CURLOPT_NOSIGNAL` | `true` | Required for resident-worker safety. |
| `CURLOPT_BUFFERSIZE` | `8_192` | Chunked streaming via `CURLOPT_WRITEFUNCTION`. |

## Streaming model

Header callback (`CURLOPT_HEADERFUNCTION`) parses the status line and headers as libcurl emits them. Body callback (`CURLOPT_WRITEFUNCTION`) writes each chunk directly into a PSR-7 stream (typically `php://temp`-backed). The full body never lives in PHP memory; it streams from libcurl into the stream and out to the caller.

## Exception hierarchy

| Class | Extends | Implements (PSR-18) | Thrown when |
| :--- | :--- | :--- | :--- |
| `HttpClientException` | `RuntimeException` | `ClientExceptionInterface` | cURL handle could not be allocated. |
| `NetworkException` | `HttpClientException` | `NetworkExceptionInterface` | Transport failure: DNS, connect timeout, read timeout, TLS error, peer reset, partial-file, send/recv error. |
| `RequestException` | `HttpClientException` | `RequestExceptionInterface` | Protocol-level failure (malformed URL, empty response) or any non-network cURL error. |

The client uses libcurl errno → PSR-18 mapping in `Client::mapCurlError()`. The full list of "network" errnos: `CURLE_COULDNT_RESOLVE_PROXY`, `CURLE_COULDNT_RESOLVE_HOST`, `CURLE_COULDNT_CONNECT`, `CURLE_OPERATION_TIMEOUTED`, `CURLE_SSL_CONNECT_ERROR`, `CURLE_GOT_NOTHING`, `CURLE_SEND_ERROR`, `CURLE_RECV_ERROR`, `CURLE_PARTIAL_FILE`.

Everything else maps to `RequestException`.

## Usage

```php
use Waffle\Commons\HttpClient\Client;
use Nyholm\Psr7\Factory\Psr17Factory;

$psr17 = new Psr17Factory();
$client = new Client(
    responseFactory: $psr17,
    streamFactory:   $psr17,
);

$request = $psr17->createRequest('GET', 'https://api.example.com/users');
$response = $client->sendRequest($request);

echo $response->getStatusCode();    // 200
echo (string) $response->getBody(); // streamed body
```

## Worker-mode safety

- The `\CurlHandle` is reused across requests. `curl_reset()` clears per-request state without freeing the handle.
- No instance state lives between `sendRequest()` calls beyond the handle itself.
- The 10-second hardcoded ceiling prevents a hung remote endpoint from locking a FrankenPHP worker.

## PHP 8.5 features used

- `final readonly class Client` with promoted constructor properties.
- Typed integer constants for `CONNECT_TIMEOUT_MS`, `TIMEOUT_MS`, `CHUNK_SIZE`.
- `#[\Override]` on PSR-18 implementation.
- `match` for HTTP-version negotiation.

## Related

- [http.md](http.md) — the PSR-7/17 implementation that produces the requests this client sends.
- [explanation/security-csrf-double-submit.md](../explanation/security-csrf-double-submit.md) — CSRF protection covers inbound; this client covers outbound.
