# Tutorial: Your First Secured CRUD Endpoint (`0.1.0-beta5`)

In this lesson you will build, from an empty directory, a `POST /notes` endpoint that is **denied by default**, then progressively earn access to it the Waffle way: an ABAC voter, a CSRF token, validated input via PHP 8.5 property hooks, and finally a database write that runs inside a pooled, failsafe transaction.

By the end you will have *seen* — not just read — Waffle's three security reflexes: **fail-closed authorization**, **cryptographically bound CSRF**, and **transactional writes on a pinned pooled connection**.

**You will need:** PHP 8.5+, Composer 2.x, Docker with Docker Compose, and (optionally) `jq` for the shell snippets. Expect 45–60 minutes.

## 1. Create the project

```bash
composer create-project waffle-commons/skeleton my-notes
cd my-notes
```

Do not start the containers yet — the skeleton expects two backing services we add first.

## 2. Add the backing services (Redis + MySQL)

The skeleton's `config/app.yaml` configures a **Redis** PSR-16 cache (used by the router) and a **MySQL** database (RFC-022), and `.env` already points at them (`REDIS_DSN=redis://waffle-redis:6379`, `DB_*`). If your `docker-compose.yml` does not already define these services, add them alongside the `php` service:

```yaml
# docker-compose.yml (add under `services:`)
  waffle-redis:
    image: redis:7-alpine
    container_name: waffle-redis

  waffle-mysql:
    image: mysql:8.4
    container_name: waffle-mysql
    environment:
      MYSQL_DATABASE: waffle
      MYSQL_USER: waffle
      MYSQL_PASSWORD: waffle
      MYSQL_ROOT_PASSWORD: waffle
    volumes:
      - mysql-data:/var/lib/mysql

# …and at the top level of the file:
volumes:
  mysql-data:
```

Point the database host at the new service in `.env` (the compose network resolves service names):

```dotenv
DB_HOST=waffle-mysql
```

Now boot everything:

```bash
docker compose up -d
```

**Expected result:** `docker compose ps` shows the `php`, `waffle-redis`, and `waffle-mysql` containers running. The app serves on `https://localhost/` (FrankenPHP terminates TLS with a self-signed certificate, hence `-k` on every `curl` below).

## 3. First contact: a public route

Create `src/Controller/NoteController.php` with a ping action. Two things matter here: routes are declared with the `#[Route]` attribute, and a public endpoint must **say so** with `#[PublicAccess]` — Waffle's ABAC is fail-closed, so an unmarked action is denied, not allowed.

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Psr\Http\Message\ResponseInterface;
use Waffle\Commons\Contracts\Routing\Attribute\Route;
use Waffle\Commons\Contracts\Routing\Constant as Routing;
use Waffle\Commons\Contracts\Security\Attribute\PublicAccess;
use Waffle\Core\BaseController;

#[Route(path: '/', name: 'notes_')]
final class NoteController extends BaseController
{
    #[Route(path: 'notes/ping', methods: [Routing::METHOD_GET], name: 'ping')]
    #[PublicAccess]
    public function ping(): ResponseInterface
    {
        return $this->jsonResponse(data: ['pong' => true]);
    }
}
```

```bash
curl -sk https://localhost/notes/ping
```

**Expected result:**

```json
{"pong":true}
```

## 4. Give the notes a table

Drop a migration into `migrations/` — the filename is the version key, applied in lexicographic order:

```sql
-- migrations/Version2026080102_CreateNotesTable.sql
CREATE TABLE IF NOT EXISTS notes (
    id VARCHAR(36) PRIMARY KEY,
    title VARCHAR(120) NOT NULL,
    body TEXT NOT NULL,
    author VARCHAR(64) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Run the forward-only migration runner from inside the container:

```bash
docker compose exec php php bin/waffle db:migrate
```

**Expected result:** each pending version is printed as it is applied (the skeleton ships a `users` migration, so a fresh database applies two), ending with:

```
Done — 2 migration(s) applied.
```

Run it again — already-applied versions are recorded in `waffle_migrations` and skipped: `Database is already up to date — nothing to apply.`

## 5. Shape the input: a DTO that validates itself

Waffle has no validation framework. Validation is **domain logic that lives on the value**, written as PHP 8.5 **property hooks**. Create `src/Dto/NoteInput.php`:

```php
<?php

declare(strict_types=1);

namespace App\Dto;

use Waffle\Commons\Contracts\Attribute\Dto;
use Waffle\Commons\Utils\Assert;
use Waffle\Exception\ValidationException;

#[Dto]
final class NoteInput
{
    // Full `set` hook: a bespoke rule, with a `field` key for the RFC 7807 payload.
    // (`private(set)` keeps the DTO externally immutable — a `set` hook cannot
    // live on a `readonly` property, so asymmetric visibility does both jobs.)
    public private(set) string $title {
        set(string $value) {
            $clean = mb_trim($value);

            if ($clean === '' || mb_strlen($clean) > 120) {
                throw new ValidationException(
                    message: 'Field "title" must be a non-empty string of at most 120 characters.',
                    field: 'title',
                );
            }

            $this->title = $clean;
        }
    }

    // Short hook: `Assert` validates AND returns the cleansed value in one line.
    public private(set) string $body {
        set => Assert::length(Assert::notEmpty($value), 1, 2000);
    }

    public function __construct(string $title, string $body)
    {
        $this->title = $title;
        $this->body = $body;
    }
}
```

When a controller parameter is type-hinted with a `#[Dto]` class, the argument resolver decodes the JSON body, maps keys to constructor parameters by name, and instantiates it — the hooks fire **before your action runs**. Invalid input never reaches you.

## 6. The write endpoint — and your first 403

Add the `create` action to `NoteController`. Write it *complete* — DTO input, identity lookup, pooled INSERT — but attach **no security attribute yet**:

```php
use App\Dto\NoteInput;
use Psr\Http\Message\ServerRequestInterface;
use Waffle\Commons\Contracts\Auth\Constant as AuthConstant;
use Waffle\Commons\Contracts\Auth\UserIdentityInterface;
use Waffle\Commons\Contracts\Data\Connection\RelationalConnectionPoolInterface;

    #[Route(path: 'notes', methods: [Routing::METHOD_POST], name: 'create')]
    public function create(
        NoteInput $input,
        ServerRequestInterface $request,
        RelationalConnectionPoolInterface $pool,
    ): ResponseInterface {
        $identity = $request->getAttribute(AuthConstant::REQUEST_ATTRIBUTE);
        $author = $identity instanceof UserIdentityInterface ? $identity->subject : 'anonymous';

        $id = bin2hex(random_bytes(18)); // 36 hex chars — fits the VARCHAR(36) key

        // The TransactionIsolationMiddleware has already pinned ONE pooled
        // connection and opened a transaction for this write request; acquire()
        // returns that same lease, so this INSERT commits on normal return and
        // rolls back on any uncaught exception.
        $statement = $pool->acquire()->pdo()->prepare(
            'INSERT INTO notes (id, title, body, author) VALUES (?, ?, ?, ?)',
        );
        $statement->execute([$id, $input->title, $input->body, $author]);

        return $this->jsonResponse(data: ['id' => $id, 'title' => $input->title, 'author' => $author], status: 201);
    }
```

Now try it:

```bash
curl -sk -X POST https://localhost/notes \
  -H 'Content-Type: application/json' \
  -d '{"title":"First note","body":"Hello, Waffle."}'
```

**Expected result: HTTP `403`.** An RFC 7807 body whose detail (visible in dev; masked in production) reads like:

```
Security Policy Violation: App\Controller\NoteController::create declares no #[Voter]
and is not marked #[PublicAccess]. Add a Voter or explicitly opt out with #[PublicAccess].
```

This is **fail-closed ABAC**: an action with no explicit policy is a missing decision, and a missing decision is a *deny*. Forgetting security cannot silently publish an endpoint.

## 7. Authenticate the caller

A voter decides *who* may write notes — so first the request needs a *who*. The skeleton wires the Universal Authentication Bridge (RFC-021), which verifies JWT Bearer tokens against `waffle.auth.secret`. Give it a real secret in `.env` (in dev, a missing secret falls back to an ephemeral per-process one — fine for booting, useless for issuing tokens you can verify):

```bash
echo "WAFFLE_AUTH_SECRET=$(openssl rand -base64 32)" >> .env
docker compose restart php
```

Then create `src/Controller/TokenController.php` — a self-contained issuer of short-lived demo JWTs. (A real application delegates issuance to an identity provider; this route exists so the lesson needs no external IdP.)

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Psr\Http\Message\ResponseInterface;
use Waffle\Commons\Contracts\Config\ConfigInterface;
use Waffle\Commons\Contracts\Routing\Attribute\Route;
use Waffle\Commons\Contracts\Routing\Constant as Routing;
use Waffle\Commons\Contracts\Security\Attribute\PublicAccess;
use Waffle\Core\BaseController;

#[Route(path: '/', name: 'token_')]
final class TokenController extends BaseController
{
    #[Route(path: 'auth/token', methods: [Routing::METHOD_POST], name: 'issue')]
    #[PublicAccess]
    public function issue(ConfigInterface $config): ResponseInterface
    {
        $secret = (string) $config->getString('waffle.auth.secret');
        $now = time();

        $claims = [
            'sub' => 'demo-user',
            'email' => 'demo@waffle.dev',
            'roles' => ['ROLE_DEMO'],
            'iss' => $config->getString('waffle.auth.jwt.issuer') ?? 'https://waffle-dev.local',
            'aud' => $config->getString('waffle.auth.jwt.audience') ?? 'waffle-skeleton',
            'iat' => $now,
            'exp' => $now + 300,
        ];

        // Compact JWS assembly: base64url(header).base64url(payload).base64url(HMAC).
        $encode = static fn(string $bin): string => rtrim(
            strtr(base64_encode($bin), from: '+/', to: '-_'),
            characters: '=',
        );
        $signingInput =
            $encode(json_encode(['alg' => 'HS256', 'typ' => 'JWT'], JSON_THROW_ON_ERROR))
            . '.'
            . $encode(json_encode($claims, JSON_THROW_ON_ERROR));
        $token = $signingInput . '.' . $encode(hash_hmac('sha256', $signingInput, $secret, binary: true));

        return $this->jsonResponse(data: ['token' => $token, 'type' => 'Bearer', 'expires_in' => 300]);
    }
}
```

```bash
TOKEN=$(curl -sk -X POST https://localhost/auth/token | jq -r .token)
echo "$TOKEN"
```

**Expected result:** a three-part `eyJ…` JWT. It expires after 5 minutes — re-run the command whenever a later step suddenly answers `401`.

## 8. The voter: who may write notes

Create `src/Security/NoteAuthorVoter.php`. A voter receives the request-scoped security context (`$ctx` — the verified identity, or `null` when anonymous) and the subject under decision, and returns a plain boolean:

```php
<?php

declare(strict_types=1);

namespace App\Security;

use Waffle\Commons\Contracts\Auth\SecurityContextInterface;
use Waffle\Commons\Contracts\Security\VoterInterface;

final class NoteAuthorVoter implements VoterInterface
{
    #[\Override]
    public function decide(SecurityContextInterface $ctx, mixed $subject = null): bool
    {
        // Anonymous callers have no identity, so `?->` makes this deny-by-default.
        return in_array('ROLE_DEMO', $ctx->getIdentity()?->roles ?? [], strict: true);
    }
}
```

Attach it to the action:

```php
use Waffle\Commons\Contracts\Security\Attribute\Voter;
use App\Security\NoteAuthorVoter;

    #[Route(path: 'notes', methods: [Routing::METHOD_POST], name: 'create')]
    #[Voter(name: NoteAuthorVoter::class)]
    public function create(/* … unchanged … */)
```

Try anonymously, then authenticated:

```bash
# Anonymous → the voter sees no identity → 403
curl -sk -X POST https://localhost/notes \
  -H 'Content-Type: application/json' \
  -d '{"title":"First note","body":"Hello, Waffle."}'

# Authenticated → ROLE_DEMO passes the voter → 201
curl -sk -X POST https://localhost/notes \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"title":"First note","body":"Hello, Waffle."}'
```

**Expected results:** `403` for the first call; for the second, something like:

```json
{"id":"6f1f0c4bcadb54f3f0f6e9c2d94b1a7e00aa41bd","title":"First note","author":"demo-user"}
```

And the row really is in MySQL, written inside the middleware's transaction:

```bash
docker compose exec waffle-mysql mysql -uwaffle -pwaffle waffle \
  -e 'SELECT title, author FROM notes;'
```

While you are here, watch the property hooks earn their keep — an invalid title never reaches your action:

```bash
curl -sk -X POST https://localhost/notes \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"title":"   ","body":"Hello."}'
```

**Expected result:** an RFC 7807 `422 Unprocessable Entity` whose detail is your hook's message — thrown by the DTO, rendered by the error handler.

## 9. CSRF: prove the request came from your client

The endpoint is authorized but still replayable by any site that can make the browser send the Bearer header. Mark it CSRF-protected:

```php
use Waffle\Commons\Contracts\Security\Csrf\Attribute\RequiresCsrfToken;

    #[Route(path: 'notes', methods: [Routing::METHOD_POST], name: 'create')]
    #[Voter(name: NoteAuthorVoter::class)]
    #[RequiresCsrfToken(id: 'form:notes')]
    public function create(/* … unchanged … */)
```

Re-run the authenticated `curl` from step 8. **Expected result: `403`** — the CSRF middleware runs before the voter and rejects the missing token.

Waffle's CSRF is a **stateless signed double-submit**: the token is an HMAC over a nonce, an expiry, the logical id (`form:notes`), and a *binding principal* — `auth:<subject>` when authenticated, else `anon:<WAFFLE_SID>`. Nothing is stored server-side; the same manager that issued the token can verify it from the request alone. Add an issuing action to `NoteController`:

```php
use RuntimeException;
use Waffle\Commons\Contracts\Security\Csrf\CsrfTokenManagerInterface;
use Waffle\Commons\Security\Csrf\CsrfBindingResolver;

    #[Route(path: 'csrf/notes', methods: [Routing::METHOD_GET], name: 'csrf')]
    #[PublicAccess]
    public function csrf(ServerRequestInterface $request, CsrfTokenManagerInterface $tokens): ResponseInterface
    {
        // The token is HMAC-bound to the CALLER. Fetch it with the same
        // credentials you will POST with, or the binding will not match.
        $binding = CsrfBindingResolver::resolve($request)
            ?? throw new RuntimeException('No CSRF binding: is AnonymousSessionMiddleware in the stack?');

        return $this->jsonResponse(data: ['csrf' => $tokens->issue('form:notes', $binding)->getValue()]);
    }
```

Complete the ceremony — token fetched *as* `demo-user`, spent *as* `demo-user`:

```bash
CSRF=$(curl -sk https://localhost/csrf/notes -H "Authorization: Bearer $TOKEN" | jq -r .csrf)

curl -sk -X POST https://localhost/notes \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-CSRF-Token: $CSRF" \
  -H 'Content-Type: application/json' \
  -d '{"title":"Second note","body":"Now with CSRF."}'
```

**Expected result:** `201` and a new note. Now prove the binding: replay the same `X-CSRF-Token` **without** the `Authorization` header. **Expected result: `403`** — a token minted for `auth:demo-user` is mathematically invalid for an anonymous (`anon:<sid>`) request. That asymmetry is the SEC-01 session-tossing defence, and you just watched it work.

## 10. Read it back

Finish the loop with a read endpoint (CSRF does not apply to safe methods, and reads are not wrapped in the write transaction — same pool, no pinning):

```php
use Waffle\Commons\Routing\Attribute\Argument;

    #[Route(path: 'notes/{id}', methods: [Routing::METHOD_GET], name: 'show', arguments: [
        new Argument(classType: 'string', paramName: 'id'),
    ])]
    #[Voter(name: NoteAuthorVoter::class)]
    public function show(string $id, RelationalConnectionPoolInterface $pool): ResponseInterface
    {
        $statement = $pool->acquire()->pdo()->prepare(
            'SELECT id, title, body, author FROM notes WHERE id = ?',
        );
        $statement->execute([$id]);
        $note = $statement->fetch(\PDO::FETCH_ASSOC);

        if (!is_array($note)) {
            return $this->jsonResponse(data: ['error' => 'Note not found.'], status: 404);
        }

        return $this->jsonResponse(data: $note);
    }
```

```bash
curl -sk "https://localhost/notes/<id-from-step-9>" -H "Authorization: Bearer $TOKEN"
```

**Expected result:** the stored note as JSON — and a `403` if you drop the Bearer header, because the voter guards reads too.

## What you built

| Layer | Attribute / mechanism | What you observed |
| :--- | :--- | :--- |
| Fail-closed ABAC | *(no attribute)* → `#[Voter]` / `#[PublicAccess]` | `403` until a policy existed; voter denies anonymous callers |
| Authentication | JWT Bearer via the RFC-021 bridge | `sub`/`roles` claims became the voter's `$ctx` identity |
| CSRF | `#[RequiresCsrfToken(id: 'form:notes')]` | stateless HMAC token, cryptographically bound to the caller |
| Validation | `#[Dto]` + property hooks | RFC 7807 `422` before your action ran |
| Persistence | `RelationalConnectionPoolInterface` + `TransactionIsolationMiddleware` | pooled INSERT inside one failsafe transaction |

## Where to go next

- Ownership rules (IDOR defence) and the two authorization layers: [How-To: Secure a Controller](../how-to/secure-a-controller.md)
- Real identity providers (OAuth2/OIDC, gateway assertions, API keys): [How-To: Authenticate Requests](../how-to/authentication.md)
- Why the CSRF design is stateless: [Explanation: CSRF Signed Double-Submit](../explanation/security-csrf-double-submit.md)
- Pool sizing, heal-on-lease, reset semantics: [Explanation: Memory-Resident Connection Pooling](../explanation/connection-pooling.md)
- Keep going with the next lesson: [First Steps with Async Work and Telemetry](first-async-and-telemetry.md)

> *Verified for Waffle Framework 0.1.0-beta5 running on PHP 8.5.5+.*
