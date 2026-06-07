# Log Reference (`waffle-commons/log`)

> **Release:** `0.1.0-beta3` &nbsp;|&nbsp; *No behavioural changes since Beta-1*
> **PSR Compliance:** PSR-3 (`Psr\Log\LoggerInterface`, `AbstractLogger`)

PSR-3 logger that emits JSON-formatted records to a stream. Optimised for Docker / Kubernetes deployments where logs are collected from `stdout` / `stderr`.

## Surface

| Class | Role |
| :--- | :--- |
| `Waffle\Commons\Log\StreamLogger` | PSR-3 logger writing JSON lines to a stream. |
| `Waffle\Commons\Log\Channel\LogChannel` | Final class of `public const string` channel identifiers. |

## `StreamLogger` — constructor

```php
namespace Waffle\Commons\Log;

final class StreamLogger extends AbstractLogger
{
    public function __construct(
        private(set) readonly string $streamPath  = 'php://stderr',
        private(set) readonly string $channel     = LogChannel::APP,
        private(set) readonly int    $permissions = 0o644,
    );
}
```

| Parameter | Default | Meaning |
| :--- | :--- | :--- |
| `streamPath` | `php://stderr` | Any fopen-able stream. `php://stdout`, `php://stderr`, a file path. |
| `channel` | `LogChannel::APP` | Channel name written into every record. |
| `permissions` | `0644` | UNIX mode applied if the stream is a regular file that the logger had to create. Ignored for `php://*` streams. |

The constructor opens the stream in `ab` mode (append-binary). If the stream cannot be opened, `Psr\Log\InvalidArgumentException` is thrown.

## Record shape

Every line is a single JSON object — newline-delimited, log-collector friendly:

```json
{
  "@timestamp": "2026-03-22T11:42:00+00:00",
  "level":      "info",
  "level_code": 200,
  "channel":    "app",
  "message":    "Order #42 created",
  "context":    { "order_id": 42, "user_id": 7 }
}
```

| Field | Source |
| :--- | :--- |
| `@timestamp` | `new DateTimeImmutable()->format(DateTimeInterface::ATOM)`. |
| `level` | The PSR-3 level (`debug`, `info`, `notice`, `warning`, `error`, `critical`, `alert`, `emergency`). |
| `level_code` | RFC 5424 / Monolog-compatible integer (`debug=100` … `emergency=600`). |
| `channel` | The channel passed to the constructor. |
| `message` | The interpolated message (PSR-3 placeholder substitution). |
| `context` | The full context array passed by the caller. |

If a context value is `Stringable`, it's coerced via `(string) $value`. Otherwise it's serialised as-is by `json_encode(JSON_UNESCAPED_SLASHES \| JSON_INVALID_UTF8_SUBSTITUTE \| JSON_PARTIAL_OUTPUT_ON_ERROR)`.

## `LogChannel` — predefined channels

```php
namespace Waffle\Commons\Log\Channel;

final class LogChannel
{
    public const string APP      = 'app';    // userland (controllers, services)
    public const string CORE     = 'core';   // framework internals (kernel, container, boot)
    public const string HTTP     = 'http';   // request/response/routing errors
    public const string SECURITY = 'sec';    // auth, ABAC, CSRF denials
    public const string AUDIT    = 'audit';  // compliance / critical business events
}
```

Using the constants prevents typos and keeps the channel set discoverable. Application-defined channels are still allowed — just declare them in your own constant class.

## Typical instantiation

```php
use Waffle\Commons\Log\StreamLogger;
use Waffle\Commons\Log\Channel\LogChannel;

$appLogger      = new StreamLogger();                                          // app, php://stderr
$securityLogger = new StreamLogger(channel: LogChannel::SECURITY);
$auditLogger    = new StreamLogger(streamPath: '/var/log/audit.log',
                                   channel: LogChannel::AUDIT,
                                   permissions: 0o600);
```

The skeleton's `AppKernelFactory` constructs distinct `StreamLogger` instances per channel and injects them where appropriate (`SecurityMiddleware` takes the `SECURITY` channel, the kernel itself takes `CORE`, …).

## Worker-mode safety

- The stream resource is opened once in the constructor and reused across all `log()` calls.
- No buffering at the PHP level — every record is written and (for `php://stderr` / `php://stdout`) flushed by the SAPI.
- `final` class with `readonly` constructor properties — no mutation across requests.

## PSR-3 placeholder interpolation

Standard PSR-3 interpolation is supported:

```php
$logger->info('User {id} signed in from {ip}', ['id' => 42, 'ip' => '203.0.113.7']);
// → message: "User 42 signed in from 203.0.113.7"
//   context: { "id": 42, "ip": "203.0.113.7" }
```

Placeholders are also retained in `context` for structured-log consumers (Loki, Splunk, Elasticsearch).

## Related

- [contracts.md](contracts.md) — PSR-3 `LoggerInterface` is the contract.
- [how-to/error-handling.md](../how-to/error-handling.md) — how `ErrorHandlerMiddleware` uses the logger.
