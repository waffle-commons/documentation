# Finish-Request Deferral & Concurrent Fan-Out (Beta-5)

> **Status:** shipped in `waffle-commons/async` (ASYNC-01) + `waffle-commons/http-client` (ASYNC-02), both behind contracts, since the `0.1.0-beta5` cycle (RFC-015).
> **Companion pages:** [async reference](../reference/async.md) · [How to: Defer Post-Response Work](../how-to/defer-post-response-work.md).

This page explains *why* the async surface looks the way it does — and, just as importantly, *what it is not*. For exact signatures, use the reference.

## The problem: short work that does not belong in the latency path

A request handler often has to do something *after* it has produced the answer: send a confirmation mail, write an audit row, fan a webhook out to a few subscribers. None of it changes the response body, yet on a classic request/response model it all runs *before* the response is flushed — so the user waits for work they will never see.

The temptation is to reach for "async" in the JavaScript sense: spawn a background thread, fire-and-forget. PHP under FrankenPHP does not work that way, and pretending it does is how worker-mode frameworks corrupt state. ASYNC-01 takes the honest path: keep the work on the *same* worker thread, but move it past the point where the response is already on the wire.

## Finish-request deferral, not background processing

The FrankenPHP worker loop emits the response, then runs a *finish-request* phase before it loops back to accept the next request. Waffle hangs a single drain on that phase: tasks a handler queued during the request via `TaskRunnerInterface::defer()` are executed by `TaskRunnerInterface::run()` **after the response is flushed to the client but before the worker accepts its next request**.

```
 ┌──────────── request N ────────────┐
 handler runs ─▶ defer(taskA, taskB)  │   response flushed ──▶ client
                                      │
                                      ▼   (TerminateEvent — finish-request)
                          run(): taskA, taskB drained
                                      │
                                      ▼   Container::reset() — request-scoped state cleared
 └──────────── worker accepts request N+1 ────────────┘
```

The user-perceived latency is the time-to-flush; the deferred work is paid for out of the worker's *own* time, after the client has its answer. The drain is wired as the `DeferredTaskFlushListener` on the kernel's `TerminateEvent` — the listener lives in the kernel package, not in `waffle-commons/async`, so the runner stays contracts-only and never imports a concrete event.

This buys latency, not throughput. While the worker drains deferred tasks it is **not** serving the next request. That is the whole reason the budget below exists.

## Fibers are an isolation boundary, not a thread pool

Each task runs inside its own native PHP `Fiber`. It is easy to misread that as "the task runs on a background thread" — it does not. **A Fiber is cooperative, single-threaded execution.** There is exactly one OS thread; the Fiber is a suspendable call frame on it, nothing more. Two deferred tasks never run *simultaneously*; they run one after another, on the worker thread, during finish-request.

So why a Fiber at all, if not for parallelism? **Failure isolation.** The runner wraps each task's entire Fiber lifecycle — start, the single cooperative resume, and its destruction — in a `try`/`catch`. A task that throws is logged and contained; its siblings still run, and the finish-request teardown still completes. Without the Fiber boundary one bad webhook would abort the audit write queued behind it and could break the worker's reset cascade.

The model is deliberately **not** a scheduler. `run()` starts a task's Fiber and, if it suspends, resumes it exactly once so a task that yields a single time can still finish. A task still suspended after that one resume is *abandoned with a warning* — the runner does not pump a loop waiting for long work to complete. That is by design: heavy or long-running work does not belong here at all.

## The budget is a hard ceiling, on purpose

`defer()` is bounded. Each request may queue at most `DeferredTaskRunner::DEFAULT_BUDGET` (64) tasks; the 65th throws `DeferralBudgetExceededException`, whose message says exactly what to do — *move this workload to a queue or broker*. A non-positive budget is itself a programming error and is rejected eagerly at construction with `InvalidBudgetException`, mirroring the data-layer connection pools' "must allow at least one" guard.

The ceiling is not a safety valve you are meant to tune ever upward. It is a tripwire. If a request routinely wants to defer dozens of tasks, the work is not "short post-response work" any more — it is a batch job, and a batch job belongs on a real queue with its own worker, retries, and dead-lettering (the Beta-6 `waffle-commons/queue` boundary, RFC-015 §future). Finish-request deferral trades worker throughput for perceived latency; the budget keeps that trade small and visible.

Tasks must be **self-contained, bounded, and worker-safe**: a `DeferredTaskInterface` carries everything it needs (the demo `LogAuditTask` is a `final readonly` value with its logger and a string action), so nothing leaks between requests and `wfl igor` stays at 0 KO.

## Worker safety: one piece of state, explicitly recyclable

The runner's only request-scoped state is the pending queue. It implements `ResettableInterface` **directly** (igor-php requires the direct declaration, not a transitive one) and the kernel empties it on every worker iteration, so deferred work queued in request N can never bleed into request N+1. `run()` snapshots the queue and clears it before draining, so a task that itself defers another task cannot grow the queue being drained, and the queue is empty even if the drain throws. The `DeferredTaskFlushListener` additionally wraps the whole drain in a catch-all: terminate runs after the response is emitted, so a throwable escaping the drain cannot reach the client — it would only corrupt teardown, so it is logged and swallowed.

## ASYNC-02: real concurrency, where it actually exists

Deferral is sequential. The one place Waffle does resolve work *in parallel* is **outbound HTTP**, and it does so without threads either — it leans on libcurl's multi interface, which multiplexes many sockets on one thread.

`waffle-commons/http-client`'s `Client` already drives every transfer through a `CurlMultiHandle` (parking on `curl_multi_select()` instead of blocking in `curl_exec()`). ASYNC-02 exposes that machinery as the `ConcurrentClientInterface`:

- **`sendRequests(array $requests)`** allocates one dedicated easy handle per request, registers them all on a single multi handle, and drives one shared `curl_multi_exec()` loop. The batch settles in roughly the wall-clock of the *slowest* request rather than the *sum* — three 200 ms backends finish in ~200 ms, not 600 ms. Input keys are preserved so the result lines up 1:1, and the call is fail-fast: any transfer failure (transport, protocol, or SSRF refusal) throws rather than silently returning partial results.
- **`promise(RequestInterface $request)`** hands back a non-blocking `PromiseInterface` over a single in-flight request, driven only when the caller invokes `wait()`.

This is genuine concurrency because libcurl waits on all the sockets at once — but it is still single-threaded and still bounded by the client's hardcoded 1 s connect / 10 s total timeout ceilings, so a hung legacy backend can never pin the worker. Per-request handles are created and freed within the call (an un-waited promise frees its handles from its destructor), so the only resident state is the persistent easy/multi handle and the client stays worker-safe. The same `SsrfGuard` (SEC-02) that protects `sendRequest()` resolves, validates, and pins every fanned-out target, so concurrency never widens the SSRF surface.

## Why the two halves live apart

ASYNC-01 (sequential, on-worker, post-response) and ASYNC-02 (concurrent, outbound, in-request) solve different problems and ship in different components — `waffle-commons/async` and `waffle-commons/http-client` respectively — joined only through contracts (`Contracts\Async\*`, `Contracts\HttpClient\*`). Neither is a substitute for a message queue: deferral is for short work you want *off* the response path, fan-out is for I/O you want resolved *together*. When you need durability, retries, or work that outlives the request, that is the queue's job, not theirs.

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
