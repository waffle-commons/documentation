# How to Run on Kubernetes

> **Release:** `0.1.0-beta7` &nbsp;|&nbsp; *Probes, graceful shutdown and migrations (OPS-01/02/03, RFC-023, RFC-014)*

This guide deploys an application built from the skeleton's production image so that Kubernetes routes
traffic only to ready pods, drains a pod before it stops, and migrates the schema before the new version
takes traffic. The skeleton ships every piece wired; what remains is your manifest.

## 1. Point the probes at `/healthz` and `/readyz`

`pipeline`'s `ProbeMiddleware` serves both paths:

- **`/healthz` — liveness.** `200 {"status":"ok"}` while the worker can serve a request. It never checks a
  dependency: one database blip must not restart every pod.
- **`/readyz` — readiness.** Runs the registered probes and answers `200 {"status":"ok"}` when ready,
  `503 {"status":"unavailable"}` otherwise. The public body names no dependency; the per-probe report goes
  only to `127.0.0.1` / `::1` or a bearer token, like `/waffle-metrics`.

The skeleton registers a probe for the relational pool (`db.pool`) and, when the cache adapter is `redis`,
one for the cache (`cache`). A probe named in `waffle.health.required` is critical: absent or down, the pod
is not ready.

```yaml
# config/app.yaml
waffle:
  health:
    required:
      - db.pool
```

The middleware is mounted before `TrustedHostMiddleware`, because kubelet probes carry the pod IP as the
`Host` header, and before the rate limiter, so probes are never throttled.

```yaml
          livenessProbe:
            httpGet: { path: /healthz, port: 80 }
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet: { path: /readyz, port: 80 }
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 1
```

`timeoutSeconds: 3` leaves room for each check's own timeout, as RFC-023 §3.2 asks, so a dependency that hangs
yields a 503 instead of a probe that times out. Opening a database connection gives up after
`waffle.database.connect_timeout` (`DB_CONNECT_TIMEOUT`, 2 s by default). Redis gives up after 1 s, to connect and
to read, on the skeleton's phpredis pools (rate limiter, circuit breaker) and on the cache's Predis client unless
its DSN sets its own timeouts. Kubernetes' default of 1 s would cut a `/readyz` that is still waiting for the
database.

Each `/readyz` call times every probe and exports `waffle_health_check_duration_seconds{check}` and
`waffle_health_check_up{check}` on `/waffle-metrics`.

## 2. Drain before SIGTERM

Kubernetes removes a terminating pod from its Service endpoints, but that takes a moment to reach every
proxy. During that moment the pod must still serve, and `/readyz` should already say "not ready". The
skeleton does it with a **drain marker**:

1. The pod's `preStop` hook creates `/tmp/waffle-draining` (`waffle.health.drain_marker`) and sleeps.
2. While the file exists, `/readyz` answers `503` without running any probe, and PHP keeps serving every
   other path.
3. SIGTERM follows. Caddy closes its listeners and lets the requests in flight finish within
   `grace_period` (`CADDY_GRACE_PERIOD`, 25 s by default).

```yaml
      terminationGracePeriodSeconds: 35   # > preStop sleep (5 s) + grace_period (25 s)
      containers:
        - name: app
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "touch /tmp/waffle-draining && sleep 5"]
```

If `terminationGracePeriodSeconds` is shorter than the sleep plus the grace period, the kubelet kills the
pod in the middle of the drain.

**Without a `preStop` hook** — plain Docker, Compose, ECS — set `CADDY_SHUTDOWN_DELAY` instead (for
example `5s`): after SIGTERM, Caddy keeps serving for that long and answers `/readyz` itself with the public
`503`, then closes its listeners and drains. The orchestrator's stop timeout must exceed the delay plus the
grace period; the skeleton's `docker-compose.prod.yml` sets `CADDY_SHUTDOWN_DELAY: 5s` and
`stop_grace_period: 35s`. Leave it at its `0s` default under Kubernetes, where the marker has already
flipped readiness. This path needs FrankenPHP 1.13.1 or later: 1.12.7 answered `500` to every PHP request
during the delay.

## 3. Keep the root filesystem read-only

The production image runs as a non-root user. Give it the same writable paths as
`docker-compose.prod.yml` — the drain marker in `/tmp`, Caddy's state in `/config` and `/data`, and
`var/` — and nothing else:

```yaml
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          volumeMounts:
            - { name: tmp, mountPath: /tmp }
            - { name: caddy-config, mountPath: /config }
            - { name: caddy-data, mountPath: /data }
            - { name: app-var, mountPath: /app/var }
      volumes:
        - { name: tmp, emptyDir: {} }
        - { name: caddy-config, emptyDir: {} }
        - { name: caddy-data, emptyDir: {} }
        - { name: app-var, emptyDir: {} }
```

## 4. Migrate from the release image, before the rollout

The production image ships `bin/` and `migrations/`, so a Job runs the migrations from the very image the
Deployment is about to roll out:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: app-migrate-1-0-0
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: registry.example.com/app:1.0.0
          command: ["php", "bin/waffle", "db:migrate"]
          envFrom:
            - secretRef: { name: app }
```

`db:status` and `db:rollback --steps=N` run the same way. Write migrations that the previous version of the
code can still run against, so the old pods keep working while the new ones start — see
[How to: Run Database Migrations](database-migrations.md).

## 5. Feed it its configuration

Everything environment-specific comes from the environment, never from the image: `APP_ENV=prod`,
`APP_DEBUG=0`, `WAFFLE_CSRF_SECRET` and `WAFFLE_AUTH_SECRET` (32 bytes or more, or the boot is refused in
production), `DB_*`, `REDIS_DSN`, `TRUSTED_HOSTS`. Keep secrets in a Secret, as the Job above does.

## Related

- [Reference: Pipeline — orchestrator probes](../reference/pipeline.md#orchestrator-probes-ops-0102)
- [Reference: Contracts — health checks](../reference/contracts.md#health-checks-ops-01)
- [Reference: Configuration Keys](../reference/configuration-keys.md)
- [How to: Rate-Limit Requests](rate-limit-requests.md)
