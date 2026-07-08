# HTTP Client Reference (`waffle-commons/http-client`)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; SEC-02 SSRF guard (resolve → validate → pin)
> **PSR Compliance:** PSR-18 (`Psr\Http\Client\ClientInterface`), consumes PSR-7 messages, PSR-17 factories

A high-performance PSR-18 HTTP client tuned for FrankenPHP resident-worker proxying. Holds a persistent `\CurlHandle` and `\CurlMultiHandle`, reused via `curl_reset()` across every `sendRequest()` so libcurl's DNS cache and keep-alive pool stay warm. The transfer is driven through the multi interface (`curl_multi_exec` + `curl_multi_select`) rather than the blocking `curl_exec()`, so the worker parks on a socket-level wait instead of busy-spinning. Bodies stream in 8 KiB chunks **in both directions** — request bodies are pulled from the PSR-7 request stream, response bodies pushed into a PSR-7 stream — so neither is materialised whole in worker memory.

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
        private ?SsrfGuard               $ssrfGuard = null, // SEC-02: opt-in SSRF defence
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

## SEC-02 — SSRF guard (resolve → validate → pin)

When a `Waffle\Commons\HttpClient\Security\SsrfGuard` is injected, every `sendRequest()` resolves the target host, asserts that **every** resolved IP is publicly routable, and pins the vetted address into the transport via `CURLOPT_RESOLVE` — *before* any socket is opened. A non-public resolution throws `SsrfException` (a PSR-18 `RequestExceptionInterface`), fail-closed.

```php
use Waffle\Commons\HttpClient\Client;
use Waffle\Commons\HttpClient\Security\SsrfGuard;

$client = new Client($psr17, $psr17, new SsrfGuard()); // default SystemHostResolver
```

- **Validation:** `Assert::isPublicIp()` rejects loopback / RFC 1918 / RFC 4193 / link-local / CGNAT / multicast / reserved ranges (IPv4 + IPv6, incl. IPv4-mapped). If *any* resolved IP is non-public the request is refused — defeating a DNS-rebinding record that mixes a public and a private answer.
- **Pinning (TOCTOU defence):** the vetted IP is pinned with `CURLOPT_RESOLVE` so libcurl reuses exactly the address that was validated — closing the time-of-check/time-of-use gap between the DNS check and the connection. Combined with `CURLOPT_FOLLOWLOCATION => false` (redirects never auto-followed), there is no unvalidated hop.
- **Host resolution** is abstracted behind `HostResolverInterface`; the default `SystemHostResolver` uses `gethostbynamel()` and short-circuits literal IPs. Supply a caching / DoH resolver if needed.
- **Opt-in:** the guard is **not** enabled by default — the framework's own internal service-to-service calls (e.g. the auth bridge's outbound assertion propagation) legitimately target private addresses. Enable it on clients that dispatch **user-influenced** URLs.

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
| `CURLOPT_BUFFERSIZE` | `8_192` | Chunk size for response streaming via `CURLOPT_WRITEFUNCTION`. |
| `CURLOPT_UPLOAD` + `CURLOPT_READFUNCTION` | set when a body is present | Streams the request body in `CHUNK_SIZE` chunks instead of buffering it with `CURLOPT_POSTFIELDS`. |
| `CURLOPT_INFILESIZE` | body length, when known | Advertises `Content-Length`; omitted for unknown-length streams (→ `Transfer-Encoding: chunked`). |

`SELECT_TIMEOUT` (private, `1.0`s) bounds each `curl_multi_select()` park.

## Transfer model (non-blocking)

`execute()` adds the easy handle to the persistent multi handle, then loops `curl_multi_exec()` / `curl_multi_select()` until the transfer completes — the worker waits on the socket set, never busy-spinning a CPU and never blocking inside `curl_exec()`. The per-transfer result is read from `curl_multi_info_read()`; a `CURLM_*`-level failure maps to `NetworkException`. The easy handle is removed from the multi handle in a `finally` block, but both handles persist across requests so keep-alive survives. The multi handle is also the foundation for future concurrent fan-out.

## Streaming model

**Request body.** When the PSR-7 request carries a body, `applyRequestBody()` enables `CURLOPT_UPLOAD` and registers a `CURLOPT_READFUNCTION` that pulls `CHUNK_SIZE` bytes at a time from the request stream — a large multipart/chunked upload is forwarded without ever buffering the whole payload (unlike `CURLOPT_POSTFIELDS`). The request method set via `CURLOPT_CUSTOMREQUEST` is preserved. Known body lengths use `CURLOPT_INFILESIZE`; unknown lengths fall back to `Transfer-Encoding: chunked`.

**Response.** The header callback (`CURLOPT_HEADERFUNCTION`) parses the status line and headers as libcurl emits them. The body callback (`CURLOPT_WRITEFUNCTION`) writes each chunk directly into a PSR-7 stream (typically `php://temp`-backed). The full body never lives in PHP memory; it streams from libcurl into the stream and out to the caller.

## Exception hierarchy

| Class | Extends | Implements (PSR-18) | Thrown when |
| :--- | :--- | :--- | :--- |
| `HttpClientException` | `RuntimeException` | `ClientExceptionInterface` | cURL handle could not be allocated. |
| `NetworkException` | `HttpClientException` | `NetworkExceptionInterface` | Transport failure: DNS, connect timeout, read timeout, TLS error, peer reset, partial-file, send/recv error. |
| `RequestException` | `HttpClientException` | `RequestExceptionInterface` | Protocol-level failure (malformed URL, empty response) or any non-network cURL error. |
| `SsrfException` | `HttpClientException` | `RequestExceptionInterface` | The SSRF guard refused the target host (missing, unresolvable, or a non-public IP). |

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

- The `\CurlHandle` and `\CurlMultiHandle` are reused across requests. `curl_reset()` clears per-request state on the easy handle without freeing it; the easy handle is added to / removed from the multi handle each call.
- No instance state lives between `sendRequest()` calls beyond the two handles (the class is `final readonly`).
- Driving the transfer through `curl_multi_select()` means the worker waits on the socket rather than blocking in `curl_exec()`; combined with the 10-second hardcoded ceiling, a hung remote endpoint can never lock a FrankenPHP worker.

## PHP 8.5 features used

- `final readonly class Client` with promoted constructor properties.
- Typed integer constants for `CONNECT_TIMEOUT_MS`, `TIMEOUT_MS`, `CHUNK_SIZE`.
- `#[\Override]` on PSR-18 implementation.
- `match` for HTTP-version negotiation.

## Related

- [http.md](http.md) — the PSR-7/17 implementation that produces the requests this client sends.
- [explanation/security-csrf-double-submit.md](../explanation/security-csrf-double-submit.md) — CSRF protection covers inbound; this client covers outbound.
