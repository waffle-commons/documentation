# How to Run Database Migrations

Waffle ships a lightweight, forward-only SQL migration runner (RFC-022) in the `waffle-commons/data` component, wired into the console as `bin/waffle db:migrate`. It applies versioned `*.sql` scripts in order and records each one in a `waffle_migrations` table so re-runs are no-ops. There is no schema diffing and no DSL — you write plain SQL.

## 1. Configure database access

Add a `database` block to `config/app.yaml`. Credentials come from the environment via `%env(...)%` placeholders:

```yaml
waffle:
  database:
    driver: 'mysql'
    host: '%env(DB_HOST)%'
    port: '%env(DB_PORT)%'
    database: '%env(DB_NAME)%'
    username: '%env(DB_USER)%'
    password: '%env(DB_PASSWORD)%'
    charset: 'utf8mb4'
    migrations_path: 'migrations'   # relative to the project root
```

…and set the values in `.env` (local) / your orchestrator (production):

```dotenv
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=waffle
DB_USER=waffle
DB_PASSWORD=waffle
```

> The skeleton and workspace templates already include this block. See [How to Manage Configuration](configuration.md) for the `%env()%` resolution and the OS-vs-`.env` precedence rule.

## 2. Write a migration file

Drop a `.sql` file into the `migrations/` directory. The filename **is** the version key, so use a sortable, unique prefix — the convention is `Version<YYYYMMDDNN>_<Description>.sql`:

```sql
-- migrations/Version2026053101_CreateUsersTable.sql
CREATE TABLE IF NOT EXISTS users (
    id VARCHAR(36) PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Migrations are applied in **lexicographic order of filename**, so the timestamp prefix determines sequencing. The version recorded in `waffle_migrations` is the filename without its `.sql` extension (e.g. `Version2026053101_CreateUsersTable`).

## 3. Run the migrations

From the project root (so the relative `migrations_path` resolves):

```bash
# inside the container
docker exec -w /waffle-commons/your-app waffle-dev php bin/waffle db:migrate

# or, locally
php bin/waffle db:migrate
```

The command prints each applied version in real time and a summary:

```
Applying database migrations…
  applied  Version2026053101_CreateUsersTable
Done — 1 migration(s) applied.
```

Run it again and already-applied migrations are skipped:

```
Applying database migrations…
Database is already up to date — nothing to apply.
```

`db:migrate` returns `ExitCode::SUCCESS` (`0`) on success and `ExitCode::FAILURE` (`1`) if a migration fails (the error is written to stderr), so CI/CD pipelines can branch on it.

## How it works

- The runner creates `waffle_migrations (version VARCHAR(255) PRIMARY KEY, applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)` on first run.
- It borrows a connection from the `PDOConnectionPool` (RFC-022) and applies each pending migration in **its own transaction**; on failure the transaction is rolled back, the run aborts, and nothing after the failure is applied.
- When the command finishes, it calls `reset()` on the pool — rolling back any straggler transaction and clearing buffers — to honour the FrankenPHP worker statelessness mandate.

## ⚠️ MySQL and transactional DDL

The per-migration transaction is fully atomic on engines with **transactional DDL** (SQLite, PostgreSQL). On **MySQL**, DDL statements (`CREATE TABLE`, `ALTER TABLE`, …) trigger an *implicit commit*, so a failed DDL step **cannot be rolled back**. On MySQL, keep **one schema change per migration file** so that a failure stays recoverable by writing a forward (compensating) migration rather than relying on rollback.

## Wiring it yourself

If you are assembling a console binary by hand, the runner and command are wired explicitly (no auto-discovery):

```php
use Waffle\Commons\Data\Migration\MigrationRunner;
use Waffle\Commons\Console\Command\MigrateCommand;

$pool    = AppKernelFactory::buildConnectionPool($config); // PDOConnectionPool from waffle.database.*
$runner  = new MigrationRunner(pool: $pool, config: $config);

$app->add(new MigrateCommand(runner: $runner, pool: $pool));
```

See the [data reference](../reference/data.md) for the `MigrationRunner` API and the [console reference](../reference/console.md) for the command surface.
