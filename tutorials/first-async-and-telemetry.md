# Tutorial: First Steps with Async Work and Telemetry (`0.1.0-beta5`)

In this lesson you will take work **off the user-perceived latency path** with Waffle's finish-request deferral (`waffle-commons/async`), then make the resident worker **observable**: you will scrape the Prometheus `/waffle-metrics` endpoint, watch its counters grow, and read the memory and pool gauges that prove the worker stays flat between requests.

**You will need:** a running skeleton project with the Redis service wired — steps 1–3 of [Your First Secured CRUD Endpoint](first-secured-crud-endpoint.md) get you there (the database is not needed for this lesson). Expect 30–45 minutes.

## 1. The problem, felt first

Some handler work does not change the response: an audit line, a webhook, a confirmation mail. Done inline, the user waits for it anyway. Waffle's answer is **finish-request deferral**: queue the work during the request, run it *after* the response has been flushed to the client — on the same worker, before it accepts the next request. Not a background queue; a latency trick with strict rules.

Keep a second terminal streaming the worker's logs for the whole lesson:

```bash
docker compose logs -f php
```

## 2. Write a deferred task

A task implements `Waffle\Commons\Contracts\Async\DeferredTaskInterface`: a `run()` that does the work and a `name()` used in the runner's failure logs. It must be **self-contained and worker-safe** — it carries everything it needs, so make it `final readonly`. Create `src/Async/BuildReportTask.php`:

```php
<?php

declare(strict_types=1);

namespace App\Async;

use Psr\Log\LoggerInterface;
use Waffle\Commons\Contracts\Async\DeferredTaskInterface;

final readonly class BuildReportTask implements DeferredTaskInterface
{
    public function __construct(
        private LoggerInterface $logger,
        private string $reportId,
    ) {}

    #[\Override]
    public function run(): void
    {
        // Stand-in for short post-response work (audit write, webhook, mail).
        // The sleep exists ONLY so you can observe the timing in this lesson —
        // real deferred tasks should be much shorter than this.
        sleep(1);
        $this->logger->info("[report] built after the response was sent: {$this->reportId}");
    }

    #[\Override]
    public function name(): string
    {
        return 'tutorial.report';
    }
}
```

## 3. Defer it from a handler

The skeleton's `AppKernelFactory` already registers the runner under the `TaskRunnerInterface` contract and subscribes its flush listener to `TerminateEvent` — so a controller only needs to inject the contract and call `defer()`. Create `src/Controller/ReportController.php`:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Async\BuildReportTask;
use Psr\Http\Message\ResponseInterface;
use Waffle\Commons\Contracts\Async\TaskRunnerInterface;
use Waffle\Commons\Contracts\Routing\Attribute\Route;
use Waffle\Commons\Contracts\Routing\Constant as Routing;
use Waffle\Commons\Contracts\Security\Attribute\PublicAccess;
use Waffle\Commons\Log\Channel\LogChannel;
use Waffle\Commons\Log\StreamLogger;
use Waffle\Core\BaseController;

#[Route(path: '/', name: 'reports_')]
final class ReportController extends BaseController
{
    #[Route(path: 'reports/run', methods: [Routing::METHOD_GET], name: 'run')]
    #[PublicAccess]
    public function run(TaskRunnerInterface $runner): ResponseInterface
    {
        $runner->defer(new BuildReportTask(
            logger: new StreamLogger(channel: LogChannel::APP),
            reportId: 'daily-' . date('Y-m-d'),
        ));

        // The task has only been QUEUED — nothing has run yet.
        return $this->jsonResponse(data: [
            'deferred' => true,
            'pending' => $runner->pending(),
        ]);
    }
}
```

## 4. Watch the response beat the work

Time the request:

```bash
curl -sk -w '\ntime_total: %{time_total}s\n' https://localhost/reports/run
```

**Expected result:** the response arrives in a few *milliseconds* —

```json
{"deferred":true,"pending":1}
```

— even though the task sleeps for a full second. Roughly one second **later**, the `[report] built after the response was sent: daily-…` line appears in your log terminal. That is the whole feature in one observation: the client got its answer, *then* the worker did the deferred work in an isolated `Fiber`, and only afterwards accepted its next request.

Three rules keep this honest:

- **Budget.** Each request may defer at most 64 tasks; the 65th `defer()` throws a `DeferralBudgetExceededException`. That exception is a design signal, not a limit to raise: a request deferring dozens of tasks is a batch job, and batch jobs belong in a real queue.
- **Isolation.** A task that throws is logged and skipped — siblings still run, and nothing reaches the client (the response is long gone).
- **Reset.** The runner is request-scoped and `Resettable`: the kernel empties it between worker iterations, so deferred work never bleeds from one request into the next.

## 5. Scrape `/waffle-metrics` — and meet a fail-closed 404

The skeleton also wires `waffle-commons/telemetry`: an SDK-free Prometheus endpoint served by `MetricsMiddleware` at the very front of the pipeline. Try it from your host machine:

```bash
curl -sk https://localhost/waffle-metrics
```

**Expected result: HTTP `404`.** Not `401` — the endpoint is **fail-closed**: a scrape is served only from an allow-listed address (the skeleton wires `127.0.0.1` / `::1`) or with a configured bearer token; anything else gets a `404` so an unauthorized caller cannot even confirm the endpoint exists.

Your host request reaches the server from the Docker bridge address, not loopback — so scrape from *inside* the container, where you are `127.0.0.1`:

```bash
docker compose exec php curl -sk https://localhost/waffle-metrics
```

**Expected result:** Prometheus text exposition, along these lines:

```
# HELP waffle_memory_usage_bytes Current real memory used by the PHP worker, in bytes.
# TYPE waffle_memory_usage_bytes gauge
waffle_memory_usage_bytes 12582912
# TYPE waffle_http_requests_total counter
waffle_http_requests_total{method="GET",status="200"} 3
# TYPE waffle_gc_runs_total counter
waffle_gc_runs_total 1
```

## 6. Read the gauges like an operator

Generate some traffic, then scrape again and compare:

```bash
for i in $(seq 1 25); do curl -sko /dev/null https://localhost/reports/run; done
docker compose exec php curl -sk https://localhost/waffle-metrics
```

What to look for, and why each one exists:

| Metric | Kind | What it tells you |
| :--- | :--- | :--- |
| `waffle_http_requests_total{method,status}` | counter | Grew by your 25 requests. Lives in **APCu shared memory**, not on the worker heap — one scrape aggregates every worker, and counters survive across requests without violating worker statelessness. |
| `waffle_http_request_duration_seconds` | summary | Request latency as observed by `TracingMiddleware`. Note your deferred task's `sleep(1)` does **not** show up here — deferral is off the measured path too. |
| `waffle_memory_usage_bytes` / `waffle_memory_peak_bytes` | gauge | Worker memory, sampled at scrape time. Scrape, send traffic, scrape again: it stays **flat**. That is the `ΔM = 0` invariant — the runtime cousin of the static `wfl igor` audit. |
| `waffle_gc_runs_total` / `waffle_gc_collected_total` / `waffle_gc_roots` | counter / gauge | PHP's cycle collector. A steadily climbing `gc_collected_total` under flat traffic is the classic smell of cyclic garbage churn. |
| `waffle_db_pool_active` / `waffle_db_pool_idle` / `waffle_db_pool_capacity` | gauge | The RFC-022 connection pool. In this lesson they sit at zero (no database traffic); after the CRUD tutorial's writes they show leases being taken and returned. `active` pinned at capacity is your leak alarm. |

## 7. (Optional) allow a real Prometheus in

The in-container scrape is fine for a lesson; a real Prometheus scrapes over the network with a **bearer token**. The skeleton passes `null` as the token — open `src/Factory/AppKernelFactory.php`, find the `MetricsMiddleware` block, and source a token from the process environment:

```php
$stack->add(
    middleware: new MetricsMiddleware(
        new PrometheusExporter($collectors),
        $responseFactory,
        $streamFactory,
        getenv('WAFFLE_METRICS_TOKEN') ?: null,   // was: null
        ['127.0.0.1', '::1'],
    ),
);
```

Set the variable in the `php` service's `environment:` block in `docker-compose.yml` (process env — the skeleton's `.env` loader deliberately does not mutate the process environment), restart, and scrape from the host:

```bash
docker compose up -d php
curl -sk -H "Authorization: Bearer <your-token>" https://localhost/waffle-metrics
```

**Expected result:** `200` with the metrics text — and still `404` without the header. The comparison is constant-time (`hash_equals`), and the IP allow-list keeps working alongside the token.

## What you built

- A `DeferredTaskInterface` task and a handler that defers it — response in milliseconds, work done after the flush, verified in the logs.
- A working Prometheus scrape of `/waffle-metrics`, including its fail-closed `404` posture from outside.
- A first operator's reading of the memory, GC, and pool gauges — and the observed `ΔM = 0` flat-memory behaviour of a resident worker.

## Where to go next

- The full deferral recipe (wiring by hand, budget handling, concurrent HTTP fan-out): [How-To: Defer Post-Response Work](../how-to/defer-post-response-work.md)
- Distributed tracing with the OpenTelemetry bridge, and instrumenting the cache: [How-To: Enable Telemetry and Metrics](../how-to/enable-telemetry-and-metrics.md)
- Why a Fiber is an isolation boundary, not a thread: [Explanation: Finish-Request Deferral](../explanation/async-finish-request-deferral.md)
- Why the SDK never enters the core: [Explanation: Contract-First Observability](../explanation/observability-telemetry.md)

> *Verified for Waffle Framework 0.1.0-beta6 running on PHP 8.5.6+.*
