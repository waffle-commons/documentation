# Performance Strategy

Waffle is designed so that a request pays only for its own work. The strategy rests on four pillars that compound: a resident **worker process** (no per-request bootstrap), **ahead-of-time compilation** (no first-request reflection), **memory-resident pooling** (no per-request connection handshakes), and **finish-request deferral** (no non-essential work on the latency path). Each pillar is explained below or on its companion page — and each one is only safe because of the [statelessness mandate](architecture.md#the-statelessness-mandate) (`ΔM = 0`, audited by `wfl igor`).

## FrankenPHP Integration
The `WaffleRuntime` native integration with [FrankenPHP](https://frankenphp.dev) allows the application to stay in memory between requests. This eliminates the bootstrapping overhead (Container creation, Config loading) for subsequent requests; what a request then costs is measured below, under [Measured numbers](#measured-numbers).

Because workers are long-lived, cumulative metric counters must live in APCu shared memory rather than the per-worker heap; the `GcCollector`/`MemoryCollector` then surface the memory-pressure signals described below without growing the worker. See [Contract-First Observability](observability-telemetry.md).

## Preloading
The framework structure is "Preloading-Friendly".
- The `waffle-commons/*` libraries are designed to be preloaded into Opcache shared memory.
- Using `opcache.preload` in your `php.ini` ensures that all core classes are available instantly, reducing I/O and CPU usage.

## Ahead-of-Time Compilation
Complementing OPcache preloading, [Ahead-of-Time Compilation](aot-compilation.md) (opt-in via `WAFFLE_AOT=1`) moves deterministic container-wiring and route-discovery work out of the first-request path — a reflection-free compiled container plus a serialized route trie — so the first request after a boot skips that work too.

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

These are results, not claims. The full method, the raw tables and every deviation live in
`bench/BENCH-GATE-RESULT.md` in the umbrella repository; the harness is `bench/` and each run is one command.
The figures below are the **Beta-7 series** (November 2026, FrankenPHP 1.13.2), measured on a laptop checked
idle against its power log; two earlier runs were discarded because the laptop was in use while they ran.

**Rig.** Four engines, one at a time, each pinned to 2 CPUs: **A** Waffle on the FrankenPHP worker, **B** Symfony
on php-fpm, **C** the same Symfony app on the FrankenPHP worker, and **D** that app doing A's per-request work
(trusted hosts, CORS, firewall, per-action authorisation, stateless CSRF, headers, one transaction per unsafe
request). B isolates the classic stack; C holds the runtime constant; D holds the runtime *and* the work
constant, so an A-vs-D difference is the framework's own. The load generator runs on the same laptop: every
figure is a relative comparison on this rig, not a capacity figure.

### Latency under constant arrival rate

Per rate step (p50, ms) — aggregates across a ladder are meaningless once any step saturates, so the table is
per step. Every engine held every rate with 0 errors:

| Workload | rps | A — Waffle | C — Symfony | D — Symfony, A's work |
|---|---:|---:|---:|---:|
| static JSON | 200 | 1.5 | 1.5 | 1.5 |
| static JSON | 800 | 0.8 | 0.8 | 0.9 |
| DB read | 50 | 3.4 | **2.9** | 3.0 |
| DB read | 400 | 1.4 | **1.1** | 1.2 |
| DB transaction | 100 | 3.4 | **3.0** | 3.1 |
| DB transaction | 800 | 1.8 | **1.3** | 1.5 |

"DB read" is a `SELECT` by primary key; "DB transaction" is `BEGIN` / `SELECT 1` / `COMMIT` on every engine —
a transaction boundary, not a durable write.

On JSON responses Waffle answers like a bare Symfony app on the same runtime. On database-backed requests it is
**slower** than Symfony doing the same work — by 0.3–0.4 ms at the median at 50–100 rps and by up to 0.3 ms at
higher rates, with a p99 up to 0.8 ms higher. The cause has not been isolated yet: it is a beta8 investigation.

### Memory and throughput against php-fpm

A closed-model run (8 → 128 concurrent clients, 2 minutes each, DB read) with php-fpm on a *dynamic* pool, the
configuration most favourable to it, and memory free to grow:

| Concurrency | A — Waffle | B — Symfony php-fpm | B/A |
|---:|---:|---:|---:|
| 8 | 70.4 MiB | 61.3 MiB | **0.87×** |
| 16 | 72.8 MiB | 75.5 MiB | 1.04× |
| 32 | 76.0 MiB | 101.2 MiB | 1.33× |
| 64 | 78.9 MiB | 146.4 MiB | 1.86× |
| 128 | 85.4 MiB | 240.3 MiB | **2.81×** |
| Growth per concurrent request | **+0.118 MiB** | +1.478 MiB | |

**php-fpm's memory grows 12.5× faster per concurrent request.** That slope — not a single constant — is the
honest form of the "less RAM than PHP-FPM" claim, and it is worth being precise about why:

- At **8 concurrent requests php-fpm uses *less* total RAM** (0.87×). Waffle pays a fixed floor (opcache, the
  JIT buffer, a resident worker set) up front.
- The **crossover is near 14 concurrent requests**.
- At **128 concurrent the measured advantage is 2.81×**, and it keeps widening, because one runtime allocates per
  process and the other does not.
- The advantage belongs to the **resident runtime**: Symfony on the same worker grows at Waffle's rate (+0.113 and
  +0.121 MiB per concurrent request for C and D).

Under that saturation, on 2 CPUs, php-fpm served 118.5 req/s with a 275 ms median and 1.2 % errors; the worker
engines served A 1 676, D 1 825 and C 2 029 req/s with a median near 17 ms. Waffle's throughput is 8 % below
Symfony doing the same work, consistent with its slower database path.

A blanket "5–10× less RAM" is therefore not something this project publishes: it is false at low concurrency and
only reached far beyond the measured range.

### Connection-pool behaviour under oversubscription

Measured in the Beta-6 run and not repeated since: 64 concurrent requests against an 8-connection pool — 8×
oversubscription — gave zero rejected requests, zero pool-exhaustion errors and no deadlock, and the pool returned
to idle with no leaked connection. Degradation is queueing, bounded and graceful. The throughput of that run is not
quoted as current.

### Memory stability (ΔM)

Two-hour soaks at 150 rps on the Symfony worker engines C and D, with worker recycling out of reach (about 270 000
requests per worker against a limit of 1 000 000): RSS slopes of +0.15 ± 0.11 and −1.00 ± 0.12 MiB/h, flat within
the instrument's resolution. Two hours establish no multi-hour behaviour. Waffle's own soak was part of the Beta-6
run; its endurance claim waits for the 72-hour soak of beta8, the runtime counterpart of the static `wfl igor`
audit.

> *Verified for Waffle Framework 0.1.0-beta7 running on PHP 8.5.12, FrankenPHP 1.13.2.*
