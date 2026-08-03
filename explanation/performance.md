# Performance Strategy

Waffle is designed to be one of the fastest PHP frameworks available. The strategy rests on four pillars that compound: a resident **worker process** (no per-request bootstrap), **ahead-of-time compilation** (no first-request reflection), **memory-resident pooling** (no per-request connection handshakes), and **finish-request deferral** (no non-essential work on the latency path). Each pillar is explained below or on its companion page — and each one is only safe because of the [statelessness mandate](architecture.md#the-statelessness-mandate) (`ΔM = 0`, audited by `wfl igor`).

## FrankenPHP Integration
The `WaffleRuntime` native integration with [FrankenPHP](https://frankenphp.dev) allows the application to stay in memory between requests. This eliminates the bootstrapping overhead (Container creation, Config loading) for subsequent requests, leading to sub-millisecond response times.

Because workers are long-lived, cumulative metric counters must live in APCu shared memory rather than the per-worker heap; the `GcCollector`/`MemoryCollector` then surface the memory-pressure signals described below without growing the worker. See [Contract-First Observability](observability-telemetry.md).

## Preloading
The framework structure is "Preloading-Friendly".
- The `waffle-commons/*` libraries are designed to be preloaded into Opcache shared memory.
- Using `opcache.preload` in your `php.ini` ensures that all core classes are available instantly, reducing I/O and CPU usage.

## Ahead-of-Time Compilation
Complementing OPcache preloading, [Ahead-of-Time Compilation](aot-compilation.md) (opt-in via `WAFFLE_AOT=1`) moves deterministic container-wiring and route-discovery work out of the first-request path — a reflection-free compiled container plus a serialized route trie — for sub-millisecond cold starts.

## Memory-bounded HTTP proxying (FinOps)

An edge gateway proxies traffic to slower upstreams (e.g. a legacy monolith). Doing that inside a resident worker *without* leaking memory or pinning threads is a FinOps concern: wasted RAM and workers blocked on a slow backend both cost money at scale. The `waffle-commons/http-client` (PSR-18) is engineered for exactly this workload.

- **Persistent connections.** The client holds a single `\CurlHandle` plus a `\CurlMultiHandle` for the worker's lifetime, reused via `curl_reset()` on every `sendRequest()`. libcurl's DNS cache and keep-alive pool stay warm across requests, so repeated calls to the same upstream skip the TCP/TLS handshake entirely. The database equivalent — keeping handles warm across worker iterations with a bounded, health-checked, reset-on-iteration pool — is [Memory-Resident Connection Pooling](connection-pooling.md).
- **Non-blocking transfer.** Requests are driven through the multi interface: the worker parks on `curl_multi_select()` (a socket-level wait) between `curl_multi_exec()` ticks instead of busy-spinning a CPU or blocking inside `curl_exec()`. A slow upstream can no longer pin a worker beyond the hard `10s` total-timeout ceiling (`1s` to connect).
- **Bounded memory, both directions.** Response bodies stream into a PSR-7 stream in **8 KiB** chunks (`CURLOPT_WRITEFUNCTION`); request bodies are *pulled* from the PSR-7 request stream in 8 KiB chunks (`CURLOPT_READFUNCTION` + `CURLOPT_UPLOAD`). Neither side is ever materialised whole in worker RAM — proxying a multi-gigabyte upload or download costs roughly one 8 KiB buffer, not gigabytes.

The net effect is a **fixed, predictable per-worker memory ceiling regardless of payload size**, and workers that are never held hostage by a slow backend. See the [HTTP Client reference](../reference/http-client.md) for the exact cURL options, SSRF protocol allowlist, and timeout constants.

The same libcurl multi-interface also powers Beta-5 [concurrent fan-out](async-finish-request-deferral.md#async-02-real-concurrency-where-it-actually-exists): N outbound requests complete in roughly the wall-clock of the slowest via one multi-handle loop. And [finish-request deferral](async-finish-request-deferral.md) keeps short post-response work (mail, audit, webhooks) off the user-perceived latency path entirely, draining it on `TerminateEvent` after the response is flushed.

## Memory-resident connection pooling

The database analogue of the persistent cURL handles above is [Memory-Resident Connection Pooling](connection-pooling.md) (Beta-5, DBAL-01/DBAL-02): PDO and Redis handles stay warm across worker iterations in a bounded, health-checked pool — a dead socket is healed on lease, never surfaced to the caller — and the pool's `reset()` rolls back any straggler transaction at the iteration boundary. Write requests are additionally wrapped in a single failsafe transaction on a pinned connection by `TransactionIsolationMiddleware`. The result: no per-request TCP/TLS/auth handshake against the database, with the worker-safety invariants intact.

## Measured numbers

These are results, not claims. The full method, raw data and every deviation live in
`bench/BENCH-GATE-RESULT.md`; the harness is `bench/` and each run is one command.

**Rig.** Three engines, one at a time, CPU pinned identically (2 cores): **A** Waffle on
the FrankenPHP worker, **B** Symfony on php-fpm, **C** the same Symfony app on the
FrankenPHP worker. B isolates the classic stack; C holds the runtime constant so an
A-vs-C difference is attributable to the framework rather than the runtime.

### Latency under constant arrival rate

Per rate step (p50, ms) — aggregates across a ladder are meaningless once any step
saturates, so the table is per-step:

| Workload | rps | A — Waffle worker | B — Symfony FPM | C — Symfony worker |
|---|---:|---:|---:|---:|
| static JSON | 200 | **1.5** | 2.3 | 1.5 |
| static JSON | 800 | **1.0** | 587.8 | 1.3 |
| DB read | 50 | 4.3 | 26.3 | **2.8** |
| DB read | 200 | 3.4 | 3311 | **1.9** |
| DB transaction | 100 | 4.5 | 2467 | **2.8** |
| DB transaction | 400 | 6.1 | 8185 | **3.0** |

"DB transaction" is a transaction boundary plus one trivial statement on **every**
engine — not a durable write. The public demo endpoint commits nothing, so the
Symfony baseline was aligned to do exactly the same work rather than issue an
`INSERT` the Waffle side never performs.

Against the classic stack the difference is structural: Symfony on php-fpm collapses on
database workloads between 100 and 200 rps — at 200 rps it completed 7 472 of 12 000
scheduled requests — while the worker engines hold single-digit milliseconds.

### Throughput and memory against php-fpm

Measured at matched concurrency with php-fpm on a *dynamic* pool (128 children), the
configuration most favourable to it:

| | Waffle worker | Symfony php-fpm |
|---|---:|---:|
| Throughput | **781.6 req/s** | 100.5 req/s |
| p50 | **35.0 ms** | 303.7 ms |
| RSS growth per concurrent request | **+0.105 MiB** | +1.099 MiB |

**Memory grows 10.5× slower per concurrent request.** That slope — not a single
constant — is the honest form of the "less RAM than PHP-FPM" claim, and it is worth
being precise about why:

- At **8 concurrent requests php-fpm uses *less* total RAM** (0.87×). Waffle pays a
  fixed floor (256 MB opcache, 128 MB JIT buffer) shared by a resident worker set.
- The **crossover is near 12–16 concurrent requests**.
- At **128 concurrent the measured advantage is 2.37×**, and it keeps widening, because
  one runtime allocates per process and the other does not.

A blanket "5–10× less RAM" is therefore not something this project publishes: it is
false at low concurrency and only reached far beyond the measured range.

### Connection-pool behaviour under oversubscription

Driving 64 concurrent requests against an 8-connection pool — 8× oversubscription:
**868 req/s**, p99.9 **98 ms**, zero rejected requests, zero pool-exhaustion errors, no
deadlock, and the pool returns to idle with no leaked connections. Degradation is
queueing, bounded and graceful.

### Memory stability (ΔM)

A constant-load soak on both worker engines with the request-recycle limit lifted, so a
leak cannot hide behind a worker restart: RSS drift stays **within measurement
resolution** across the run, the runtime counterpart of the static `wfl igor` audit.

> *Verified for Waffle Framework 0.1.0-beta6 running on PHP 8.5.6, FrankenPHP 1.12.2.*
