# Async Reference (`waffle-commons/async`)

> **Release:** `0.1.0-beta5` &nbsp;|&nbsp; *Fiber-based finish-request deferral (ASYNC-01, RFC-015)*
> **Requires:** PHP 8.5+, `psr/log`. Depends only on `waffle-commons/contracts`.
> **Companion:** the concurrent HTTP fan-out (ASYNC-02) ships in `waffle-commons/http-client` — see the [HTTP Client reference](http-client.md); both halves are summarised here because they share the RFC-015 async surface.

A bounded, worker-safe runner that lifts short post-response work — mail delivery, webhook fan-out, audit writes — out of the user-perceived latency path. Tasks deferred during a request run **after the response is flushed to FrankenPHP but before the worker accepts its next request**, each inside its own native `Fiber` for failure isolation. This is *finish-request deferral*, **not** background processing: the work runs on the same single worker thread, sequentially, under a hard per-request budget.

For the architectural reasoning (why a Fiber is an isolation boundary and not a thread, why the budget is a tripwire), see [Explanation: Finish-Request Deferral & Concurrent Fan-Out](../explanation/async-finish-request-deferral.md).

## Contracts (`waffle-commons/contracts`)

The whole surface is contract-first so consumers (controllers, the kernel) depend on interfaces, never the concrete runner.

### `Waffle\Commons\Contracts\Async\TaskRunnerInterface`

Request-scoped runner; **extends `Waffle\Commons\Contracts\Service\ResettableInterface`**, so the kernel resets it between worker iterations.

```php
public function defer(DeferredTaskInterface $task): void; // throws DeferralBudgetExceededExceptionInterface
public function run(): void;                               // drain + execute every pending task
public function pending(): int;                            // tasks currently queued this request
public function reset(): void;                             // (from ResettableInterface) empty the queue
```

- `defer()` enqueues a task to run at finish-request time; it throws `DeferralBudgetExceededExceptionInterface` when the per-request budget is exhausted — the signal to move the workload to a real queue.
- `run()` drains and executes every pending task, each in its own `Fiber`, isolating per-task failures so one failing task cannot abort the rest. The model is cooperative (no preemptive timeout): tasks are expected to run to completion.

### `Waffle\Commons\Contracts\Async\DeferredTaskInterface`

A single unit of deferred work. Implementations MUST be self-contained, bounded, and worker-safe.

```php
public function run(): void;     // execute the work; runs inside an isolated Fiber. @throws \Throwable
public function name(): string;  // short, stable label used in logs and telemetry
```

A thrown `\Throwable` from `run()` is caught and isolated by the runner. Long-running work belongs in a real queue/broker, not the deferral budget.

### Exception contracts

| Interface | Extends | Meaning |
| :--- | :--- | :--- |
| `Contracts\Async\Exception\AsyncExceptionInterface` | `\Throwable` | Marker for every failure originating in the task runner — catch this to stay decoupled from the concrete component. |
| `Contracts\Async\Exception\DeferralBudgetExceededExceptionInterface` | `AsyncExceptionInterface` | The per-request budget was exceeded. Exposes `budget(): int`. The explicit "move it to a real queue" signal — the budget is a hard ceiling, not advisory. |

## Concrete runner — `Waffle\Commons\Async\DeferredTaskRunner`

`final` implementation of `TaskRunnerInterface` and `ResettableInterface` (declared **directly**, as `igor-php` requires).

```php
public const int DEFAULT_BUDGET = 64;

public function __construct(
    int $budget = self::DEFAULT_BUDGET,             // hard ceiling on tasks deferred per request (>= 1)
    LoggerInterface $logger = new NullLogger(),     // PSR-3; receives per-task failure/abandon logs
);

public function defer(DeferredTaskInterface $task): void; // throws DeferralBudgetExceededException at the ceiling
public function run(): void;                               // snapshot, clear, then drain each task in its own Fiber
public function pending(): int;
public function reset(): void;                             // worker reset: empty the pending queue
```

- **Budget guard.** A `$budget` below `1` throws `InvalidBudgetException` at construction (a budget that rejects every deferral is a programming error). The `DEFAULT_BUDGET` is `64`.
- **`defer()`** appends to a request-scoped `list<DeferredTaskInterface>`; once `pending() >= $budget` it throws `DeferralBudgetExceededException` rather than enqueuing.
- **`run()`** snapshots the queue and clears it *before* draining, so a task that itself defers another task cannot grow the queue being drained, and the queue is empty even if the drain throws. Each task is then executed via an isolated `Fiber`.
- **Fiber lifecycle (per task).** The runner `start()`s the task's Fiber; if it is not terminated it is `resume()`d **exactly once** (the single cooperative resume — there is no scheduler loop). If it is *still* not terminated it is **abandoned with a `warning`** log (`… still suspended after its single cooperative resume … move long-running work to a real queue`). The Fiber is then force-destroyed (`unset`) inside the same `try`, so a throwing destructor/`finally` is caught and logged, not detonated as a fatal that would abort sibling tasks.
- **Failure isolation.** Any `\Throwable` from a task is caught and logged at `error` level (`Deferred task "{task}" failed: {message}`, with the exception attached) using `DeferredTaskInterface::name()` as the label; siblings continue.
- **`reset()`** empties the pending queue — the only request-scoped state — so deferred work never bleeds across the FrankenPHP worker boundary.

## Exceptions (`Waffle\Commons\Async\Exception\…`)

Both implement a contracts interface so callers catch deferral failures without coupling to the concrete component.

| Class | Extends / implements | Raised when |
| :--- | :--- | :--- |
| `Exception\DeferralBudgetExceededException` | `\RuntimeException` → `DeferralBudgetExceededExceptionInterface` | A request defers past its budget. Message: *"Per-request deferral budget of N task(s) exceeded; move this workload to a queue or broker."* Exposes `budget(): int`. |
| `Exception\InvalidBudgetException` | `\InvalidArgumentException` → `AsyncExceptionInterface` | The runner is constructed with a budget below `1`. Message: *"A deferral budget must allow at least one task, got N."* Exposes `budget(): int`. |

## Finish-request wiring (kernel package)

The drain is **not** in `waffle-commons/async` (which stays contracts-only). The kernel package wires it:

- `Waffle\Event\Listener\DeferredTaskFlushListener` (`final readonly`) — `__construct(TaskRunnerInterface $runner, LoggerInterface $logger = new NullLogger())`. Subscribed to `Waffle\Event\TerminateEvent` (which fires after the response is flushed, before request-scoped services reset); its `__invoke()` calls `$runner->run()` and wraps the whole drain in a catch-all (terminate runs after the response is emitted, so a throwable here cannot reach the client — it is logged and swallowed).
- The template `AppKernelFactory` registers the runner under `TaskRunnerInterface::class` with `DeferredTaskRunner::DEFAULT_BUDGET`, then adds the flush listener on `TerminateEvent` **after** the reactive broadcast flush (latency-sensitive SSE push first, then the potentially longer deferred drain).

## Concurrent HTTP fan-out (ASYNC-02 — `waffle-commons/http-client`)

The concurrency half of RFC-015. `Waffle\Commons\HttpClient\Client` (`final readonly`) implements `Waffle\Commons\Contracts\HttpClient\ConcurrentClientInterface` in addition to PSR-18's `ClientInterface`:

```php
namespace Waffle\Commons\Contracts\HttpClient;

interface ConcurrentClientInterface
{
    /** @param array<array-key, RequestInterface> $requests  @return array<array-key, ResponseInterface> */
    public function sendRequests(array $requests): array;     // throws \Psr\Http\Client\ClientExceptionInterface
    public function promise(RequestInterface $request): PromiseInterface;
}
```

- **`sendRequests()`** allocates one dedicated easy handle per request, registers them all on a single `CurlMultiHandle`, and drives one shared `curl_multi_exec()` loop, so **N requests complete in roughly the wall-clock of the slowest single request**. Array keys are preserved (1:1 with the input). Fail-fast: any transport, protocol, or SSRF failure throws (`NetworkException` / `RequestException` / `SsrfException`, all PSR-18 `ClientExceptionInterface`); partial results are never returned. Per-request handles are removed and closed in a `finally`, leaving no resident state. An empty array returns `[]`.
- **`promise()`** returns a non-blocking `PromiseInterface` over one in-flight request; the transfer is driven only when `wait()` is called.

### `Waffle\Commons\Contracts\HttpClient\PromiseInterface`

```php
public function state(): PromiseState;                // Pending | Fulfilled | Rejected
public function then(callable $onFulfilled): self;    // callable(ResponseInterface): void — settle notification
public function catch(callable $onRejected): self;    // callable(Throwable): void — settle notification
public function wait(): ResponseInterface;            // block + drive to completion; throws ClientExceptionInterface on reject
```

The callbacks are **settle notifications, not a monadic chain** — they observe the outcome and return nothing, so no `mixed` result type ever propagates. Registering `then()`/`catch()` on an already-settled promise fires the callback immediately. `Waffle\Commons\Contracts\HttpClient\Enum\PromiseState` is a `string` enum: `Pending = 'pending'`, `Fulfilled = 'fulfilled'`, `Rejected = 'rejected'` — the transition is terminal (exactly once).

The concrete `Waffle\Commons\HttpClient\Promise\Promise` (`final`, marked `#[WorkerSafe(scope: 'per-request', …)]`) runs its injected resolver exactly once and caches the outcome; repeated `wait()` calls replay the cached result. It frees its per-promise cURL handles when it settles, or from its `__destruct()` if `wait()` is never called, so an abandoned promise leaks nothing across requests. The same `SsrfGuard` (SEC-02), 1 s connect / 10 s total timeout ceilings, and chunked streaming as `sendRequest()` apply to every fanned-out transfer.

## Worker-safety contract

`DeferredTaskRunner`'s pending queue is the only request-scoped state, recyclable via `reset()`; the contracts, exceptions, and `DeferredTaskInterface` implementations are stateless / immutable, so the component passes the `igor-php` worker-mode audit with zero findings. On the fan-out side the per-promise `Promise` is the only mutable object and is a transient per-request value (marked `#[WorkerSafe]`), minted fresh per call and discarded once settled. No async state ever crosses the FrankenPHP worker boundary.

## Quick example

```php
use Psr\Log\LoggerInterface;
use Waffle\Commons\Async\DeferredTaskRunner;
use Waffle\Commons\Contracts\Async\DeferredTaskInterface;
use Waffle\Commons\Contracts\Async\TaskRunnerInterface;

// 1. A self-contained, worker-safe task.
final readonly class SendReceiptTask implements DeferredTaskInterface
{
    public function __construct(
        private LoggerInterface $logger,
        private string $orderId,
    ) {}

    #[\Override]
    public function run(): void
    {
        // short post-response work: mail / webhook / audit write
        $this->logger->info("receipt sent for {$this->orderId}");
    }

    #[\Override]
    public function name(): string
    {
        return 'order.receipt';
    }
}

// 2. In a handler — defer, return immediately, the kernel drains on TerminateEvent.
public function placeOrder(TaskRunnerInterface $runner): ResponseInterface
{
    $runner->defer(new SendReceiptTask($this->logger, $orderId)); // queued, not run yet
    return $this->jsonResponse(['pending' => $runner->pending()]);
    // ── response flushed ── then run() executes SendReceiptTask in an isolated Fiber.
}
```

For the integrator's recipe (wiring, the budget, when to reach for a real queue, and the fan-out variant) see [How to: Defer Post-Response Work](../how-to/defer-post-response-work.md).

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
