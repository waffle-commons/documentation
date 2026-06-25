# How to Defer Post-Response Work

Some work in a handler does not change the response: a confirmation mail, an audit row, a webhook fan-out. Waffle's `waffle-commons/async` component (ASYNC-01, RFC-015) lets you run that work **after the response has been flushed to the client but before the worker accepts its next request**, so the user never waits for it. Each task runs in its own isolated `Fiber`, under a bounded per-request budget.

> **This is finish-request deferral, not a background queue.** Tasks run sequentially on the same single worker thread — a `Fiber` is a cooperative isolation boundary, not a background thread. Keep deferred tasks *short*. For durable, retryable, or long-running work, use a real queue (the budget below is the tripwire that tells you when you have crossed that line).

## 1. Write a deferred task

Implement `Waffle\Commons\Contracts\Async\DeferredTaskInterface`. A task must be **self-contained, bounded, and worker-safe** — it carries everything it needs and shares no state between requests, so make it `final readonly`:

```php
<?php

declare(strict_types=1);

namespace App\Async;

use Psr\Log\LoggerInterface;
use Waffle\Commons\Contracts\Async\DeferredTaskInterface;

final readonly class LogAuditTask implements DeferredTaskInterface
{
    public function __construct(
        private LoggerInterface $logger,
        private string $action,
    ) {}

    #[\Override]
    public function run(): void
    {
        // Short post-response work: mail delivery, webhook, audit write.
        $this->logger->info("[audit] deferred action ran after response: {$this->action}");
    }

    #[\Override]
    public function name(): string
    {
        // Stable label used in the runner's failure / abandonment logs.
        return 'demo.audit';
    }
}
```

A `\Throwable` from `run()` is caught and logged by the runner — it never aborts sibling tasks and never reaches the client. Do **not** suspend the Fiber expecting it to be scheduled: the runner resumes a suspended task exactly once, then abandons it with a warning. If your work needs to yield repeatedly or run long, it belongs in a queue, not here.

## 2. Defer it from a handler

Inject `Waffle\Commons\Contracts\Async\TaskRunnerInterface` (the runner is registered in the container) and call `defer()`. The handler returns immediately; the task has only been *queued*:

```php
use Waffle\Commons\Contracts\Async\TaskRunnerInterface;

#[Route(path: 'async/audit', methods: [Routing::METHOD_POST], name: 'audit')]
public function deferAudit(TaskRunnerInterface $runner): ResponseInterface
{
    $runner->defer(new LogAuditTask(
        logger: $this->logger,
        action: 'order.shipped',
    ));

    return $this->jsonResponse([
        'deferred' => true,
        'pending'  => $runner->pending(), // tasks queued for this request
    ]);
}
```

After this handler returns, the response is flushed to the client, then the kernel drains the queue: `LogAuditTask::run()` executes in its own Fiber, after the response, off the latency path.

## 3. (Usually already done) Wire the finish-request drain

The template `AppKernelFactory` already wires this; you only do it by hand if you are assembling a kernel yourself. The runner is registered under the contract and a `DeferredTaskFlushListener` is subscribed to `TerminateEvent` (which fires after the response is flushed). Register it **after** any latency-sensitive finish-request work (e.g. the reactive SSE broadcast flush):

```php
use Waffle\Commons\Async\DeferredTaskRunner;
use Waffle\Commons\Contracts\Async\TaskRunnerInterface;
use Waffle\Event\Listener\DeferredTaskFlushListener;
use Waffle\Event\TerminateEvent;

$runner = new DeferredTaskRunner(
    budget: DeferredTaskRunner::DEFAULT_BUDGET, // 64
    logger: $logger,                            // PSR-3; receives per-task failure/abandon logs
);

$container->set(TaskRunnerInterface::class, $runner);
$listenerProvider->addListener(
    TerminateEvent::class,
    new DeferredTaskFlushListener($runner, $logger),
);
```

Because `DeferredTaskRunner` implements `ResettableInterface`, the kernel empties its queue between worker iterations — deferred work from one request never bleeds into the next.

## 4. Respect the budget — it is your "move to a queue" signal

Each request may defer at most `DeferredTaskRunner::DEFAULT_BUDGET` (**64**) tasks. The 65th `defer()` throws `Waffle\Commons\Async\Exception\DeferralBudgetExceededException`:

```
Per-request deferral budget of 64 task(s) exceeded; move this workload to a queue or broker.
```

Do not "fix" this by raising the budget. The exception is telling you the truth: a request that wants to defer dozens of tasks is no longer doing *short post-response work*, it is doing a batch job — and a batch job needs a real queue with its own worker, retries, and dead-lettering. (A budget below `1` is itself rejected at construction with `InvalidBudgetException`.)

```php
use Waffle\Commons\Contracts\Async\Exception\DeferralBudgetExceededExceptionInterface;

try {
    $runner->defer($task);
} catch (DeferralBudgetExceededExceptionInterface $e) {
    // Catch the contract, not the concrete class. $e->budget() is the ceiling that was hit.
    // Enqueue onto a durable queue instead of deferring.
}
```

## How it works

- The FrankenPHP worker emits the response, then runs a finish-request phase before looping. `DeferredTaskFlushListener` hangs on `TerminateEvent` and calls `TaskRunnerInterface::run()` there.
- `run()` snapshots and clears the queue, then runs each task inside its own native `Fiber`. The Fiber is an **isolation boundary**, not a thread: tasks run one after another on the single worker thread. A throwing task is logged at `error` level and skipped; siblings still run.
- The runner is cooperative, with no scheduler: a task that suspends is resumed exactly once, and if it is still suspended afterwards it is abandoned with a warning. Long work does not belong here.
- After the drain, the kernel calls `reset()` on the runner (and every other request-scoped service), so the worker starts the next request with an empty queue (`wfl igor` 0 KO).

## Bonus: resolve several outbound requests at once (ASYNC-02)

The *concurrency* half of RFC-015 is separate — it lives in `waffle-commons/http-client`. When a handler must call several outbound services, inject `Waffle\Commons\Contracts\HttpClient\ConcurrentClientInterface` and fan them out so the batch settles in roughly the wall-clock of the **slowest** request, not the sum:

```php
use Psr\Http\Client\ClientExceptionInterface;
use Psr\Http\Message\RequestFactoryInterface;
use Waffle\Commons\Contracts\HttpClient\ConcurrentClientInterface;

public function fanOut(ConcurrentClientInterface $client, RequestFactoryInterface $requests): ResponseInterface
{
    $batch = [];
    foreach (['http://svc-a/health', 'http://svc-b/health', 'http://svc-c/health'] as $url) {
        $batch[$url] = $requests->createRequest(Routing::METHOD_GET, $url);
    }

    try {
        // One shared multi-handle loop; keys preserved 1:1; fail-fast on any error.
        $responses = $client->sendRequests($batch);
    } catch (ClientExceptionInterface $error) {
        return $this->jsonResponse(['ok' => false, 'reason' => $error->getMessage()], status: 502);
    }

    $statuses = [];
    foreach ($responses as $url => $response) {
        $statuses[$url] = $response->getStatusCode();
    }

    return $this->jsonResponse(['ok' => true, 'statuses' => $statuses]);
}
```

For a single non-blocking request, `$client->promise($request)` returns a `PromiseInterface`; call `wait()` when you need the response (or register `then()` / `catch()` settle callbacks). The same SSRF guard and 1 s/10 s timeout ceilings apply to every fanned-out transfer — concurrency never widens the SSRF surface. This is happening *inside* the request (not deferred), and it is still single-threaded: libcurl multiplexes the sockets, it does not spawn threads.

See the [async reference](../reference/async.md) for the full API and the [explanation](../explanation/async-finish-request-deferral.md) for why a Fiber is an isolation boundary and not a thread pool.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
