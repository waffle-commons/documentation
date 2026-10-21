# How to Run Database Migrations

Waffle ships a lightweight SQL migration runner (RFC-022) in the `waffle-commons/data` component. It applies versioned `*.sql` scripts in order and records each one in a `waffle_migrations` table, so re-runs are no-ops; a version can be reverted through its down script. There is no schema diffing and no DSL — you write plain SQL.

*Reversible migrations, `db:status`, `db:rollback` and `make:migration` were added in `0.1.0-beta7` (OPS-03).* The console commands cover the whole cycle:

| Command | What it does |
| :--- | :--- |
| `bin/waffle make:migration <Name>` | Writes a new version and its down script into the migrations directory. |
| `bin/waffle db:migrate` | Applies every pending version, in version order. |
| `bin/waffle db:status` | Lists every version: applied, pending or missing. |
| `bin/waffle db:rollback [--steps=N]` | Reverts the N most recent versions (1 by default) through their down scripts. |

## 1. Configure database access

Add a `database` block to `config/app.yaml`. Credentials come from the environment via `%env(...)%` placeholders:

```yaml
waffle:
  database:
    driver: 'pgsql'                 # pgsql | mysql/mariadb (fallback when unset) | sqlite | sqlsrv | oci
    host: '%env(DB_HOST)%'
    port: '%env(DB_PORT)%'
    database: '%env(DB_NAME)%'
    username: '%env(DB_USER)%'
    password: '%env(DB_PASSWORD)%'
    charset: 'utf8'                 # ignored by pgsql (client encoding is negotiated); used by mysql/oci
    migrations_path: 'migrations'   # relative to the project root
```

`AppKernelFactory::buildDsn()` produces the engine-specific PDO DSN for each driver (PostgreSQL takes no `charset` DSN attribute; SQLite expects a file path; SQL Server and Oracle have their own grammar) and adapts the pool's liveness probe (`SELECT 1 FROM DUAL` on Oracle).

…and set the values in `.env` (local) / your orchestrator (production). The workspace template ships a ready **`waffle-postgres`** compose service (`postgres:17-alpine`, healthcheck, persistent volume), so `DB_HOST` is that service name on the internal network:

```dotenv
DB_HOST=waffle-postgres   # the compose service name; use 127.0.0.1 from the host (5432 is published)
DB_PORT=5432
DB_NAME=waffle
DB_USER=waffle
DB_PASSWORD=waffle
```

> The skeleton and workspace templates already include this block. See [How to Manage Configuration](configuration.md) for the `%env()%` resolution and the OS-vs-`.env` precedence rule.

## 2. Create a migration

From the project root (every command below runs there, so the relative `migrations_path` resolves):

```bash
php bin/waffle make:migration CreateOrdersTable
```

```
[SUCCESS] File successfully generated: /app/migrations/Version2026102101_CreateOrdersTable.sql
[SUCCESS] File successfully generated: /app/migrations/Version2026102101_CreateOrdersTable.down.sql
```

The filename **is** the version key: `Version<YYYYMMDDNN>_<Name>`. The date comes from the clock the application injects (the templates pass `SystemClock`, which reads UTC), and `NN` is the next number of that day not yet used in the directory (`01` to `99`), so versions sort in the order they were created. The name must be a PHP identifier (letters, digits and underscores), which also keeps it a safe filename. `--target=<dir>` writes somewhere else than the configured directory.

Both files start with comments only. Write the change in the first and its exact undo in the second:

```sql
-- migrations/Version2026102101_CreateOrdersTable.sql
CREATE TABLE orders (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL REFERENCES users (id),
    total_cents INTEGER NOT NULL
);
```

```sql
-- migrations/Version2026102101_CreateOrdersTable.down.sql
DROP TABLE orders;
```

A script that holds only comments, blank lines or semicolons is **refused** — `db:migrate` stops with `Migration file "…" holds no SQL statement, only comments or blank lines: write it or delete it.` and records nothing. Without that check, PostgreSQL and SQLite would run the empty script as a no-op and log the version as applied.

You can also write the files by hand: any `V.sql` is version `V`, and `V.down.sql`, when it exists, is its down script. Migrations are applied in **lexicographic order of filename**. When a version cannot be undone, delete its `.down.sql`: `db:rollback` then refuses to go past it.

## 3. Apply the pending migrations

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
  applied  Version2026102101_CreateOrdersTable
Done — 2 migration(s) applied.
```

Run it again and already-applied migrations are skipped:

```
Applying database migrations…
Database is already up to date — nothing to apply.
```

## 4. Check where each version stands

```bash
php bin/waffle db:status
```

```
STATUS   VERSION                              APPLIED AT                  DOWN
applied  Version2026053101_CreateUsersTable   2026-10-21 03:40:31.546648  yes
pending  Version2026102101_CreateOrdersTable  -                           yes
2 migration(s): 1 applied, 1 pending, 0 missing.
```

- **applied** — recorded in `waffle_migrations`, script on disk.
- **pending** — on disk, not recorded: the next `db:migrate` applies it.
- **missing** — recorded, but its script is no longer on disk (a deleted or renamed file).

`APPLIED AT` is the ledger value as the driver returns it, so its text format differs between engines. `DOWN` says whether a down script exists. `db:status` never runs a migration script; it only creates the `waffle_migrations` table when it is absent.

## 5. Roll back

```bash
php bin/waffle db:rollback             # the most recent version
php bin/waffle db:rollback --steps=2   # the two most recent, highest version first
```

```
Rolling back up to 2 migration(s)…
  reverted  Version2026102101_CreateOrdersTable
  reverted  Version2026053101_CreateUsersTable
Done — 2 migration(s) reverted.
```

Each down script runs in its own transaction, which also removes the version from `waffle_migrations` (on MySQL, MariaDB and Oracle a DDL statement commits on its own, see the caveat below), so the next `db:migrate` applies it again. **Every targeted version is checked before anything runs**: when one has no down script, an empty one, or no script on disk at all, the command reverts nothing and says which version blocks it. With nothing applied it prints `No migration is applied — nothing to roll back.`

## Exit codes

Every command returns `ExitCode::SUCCESS` (`0`) on success and writes its error to stderr otherwise, so CI/CD pipelines can branch on it:

| Command | `0` | `1` (`FAILURE`) | `64` (`USAGE`) |
| :--- | :--- | :--- | :--- |
| `db:migrate` | applied, or already up to date | a migration failed (its transaction rolled back, the run aborted) | — |
| `db:status` | listed (including an empty project) | the migration log cannot be read | — |
| `db:rollback` | reverted, or nothing applied | a targeted version cannot be reverted, or its down script failed | `--steps` is not a whole number of at least 1 |
| `make:migration` | both files written | missing or invalid name, no version left that day, or the files cannot be written | — |

## How it works

- The runner creates the `waffle_migrations` ledger (`version` primary key + `applied_at`) on first use, with the SQL of the connection's engine: `CREATE TABLE IF NOT EXISTS` on PostgreSQL, MySQL, MariaDB and SQLite, an `OBJECT_ID` guard and `DATETIME2` on SQL Server, a PL/SQL block that tolerates "table exists" on Oracle. It reads the engine from the PDO driver name.
- It borrows a connection from the `PDOConnectionPool` (RFC-022) and runs each script in **its own transaction**, together with the ledger update; on failure the transaction is rolled back, the run aborts, and nothing after the failure runs.
- When a command finishes, it calls `reset()` on the pool — rolling back any straggler transaction and clearing buffers — to honour the FrankenPHP worker statelessness mandate.
- Discovery and execution happen only on the CLI path, never during an HTTP request.

## ⚠️ Transactional DDL depends on the engine

The per-migration transaction is fully atomic on engines with **transactional DDL**: SQLite, PostgreSQL and SQL Server. On **MySQL, MariaDB and Oracle**, every DDL statement (`CREATE TABLE`, `ALTER TABLE`, …) commits implicitly, so a failed DDL step **cannot be rolled back** — in a down script as much as in an up script. On those engines, keep **one schema change per migration file**, so that a failure stays recoverable by writing a compensating migration.

The migrate → status → rollback round trip is tested live on PostgreSQL, SQLite, MySQL 8.4 and MariaDB 11.4. SQL Server and Oracle are covered by SQL-generation tests only, as no development image carries `pdo_sqlsrv` or `pdo_oci`.

## Wiring it yourself

If you are assembling a console binary by hand, the runner and the commands are wired explicitly (no auto-discovery):

```php
use Waffle\Commons\Console\Command\MigrateCommand;
use Waffle\Commons\Console\Command\MigrationRollbackCommand;
use Waffle\Commons\Console\Command\MigrationStatusCommand;
use Waffle\Commons\Console\Maker\Command\MakeMigrationCommand;
use Waffle\Commons\Data\Migration\MigrationRunner;
use Waffle\Commons\Utils\Clock\SystemClock;

$pool    = AppKernelFactory::buildConnectionPool($config); // PDOConnectionPool from waffle.database.*
$runner  = new MigrationRunner(pool: $pool, config: $config);

$app->add(new MigrateCommand(runner: $runner, pool: $pool));
$app->add(new MigrationStatusCommand(runner: $runner, pool: $pool));
$app->add(new MigrationRollbackCommand(runner: $runner, pool: $pool));
$app->add(new MakeMigrationCommand(
    clock: new SystemClock(),
    migrationsPath: $config->getString(key: 'waffle.database.migrations_path') ?? 'migrations',
));
```

The three `db:` commands depend only on contracts (`MigrationRunnerInterface`, `ResettableInterface`), so `console` never imports `data`. See the [data reference](../reference/data.md) for the `MigrationRunner` API and the [console reference](../reference/console.md) for the command surface.
