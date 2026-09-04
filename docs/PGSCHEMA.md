# pgschema

Reference notes for [pgschema](https://www.pgschema.com/) — declarative schema migration for PostgreSQL — and the design for a future set of reusable workflows in this repo.

This repo provides reusable GitHub Actions workflows and a setup composite action for pgschema. The [CLI Reference](#cli-reference) section reflects the upstream docs as of the pgschema 1.0 line.

## What It Is

pgschema is a Terraform-style declarative migration tool for PostgreSQL 14+. You keep the desired schema state in a `.sql` file, and pgschema diffs it against a live database to produce a migration plan.

pgschema is designed specifically for PostgreSQL and uses plain PostgreSQL DDL as the schema source of truth.

The fingerprint mechanism is the main reason to prefer the two-phase flow: the plan JSON embeds a hash of the database schema at plan time, and `apply` refuses to run if the database drifted since.

## Core Workflow

```
1. dump    live DB           ──► schema.sql        (baseline / bootstrap, one-time)
2. edit    schema.sql                              (desired state, in git)
3. plan    schema.sql vs DB  ──► plan.json + plan.txt
4. review  plan.txt                                (PR comment / job summary)
5. apply   plan.json         ──► DB                (fingerprint-checked)
```

Direction is always **current state (database) → desired state (file)**. The file is the source of truth.

## CLI Reference

### Installation

| Method | Command |
|--------|---------|
| Homebrew (macOS) | `brew tap pgplex/pgschema && brew install pgschema` |
| Go | `go install github.com/pgplex/pgschema@latest` (Go 1.24+) |
| Docker | `docker pull pgplex/pgschema:latest` |
| Binary | Download from [releases](https://github.com/pgplex/pgschema/releases) — `pgschema-linux-amd64`, `pgschema-1.0.0-darwin-arm64`, etc. |
| DEB / RPM | `pgschema_<ver>_amd64.deb`, `pgschema-<ver>-1.x86_64.rpm` |

Windows is not supported (use WSL). Linux AMD64/ARM64 and macOS Intel/Apple Silicon only.

For CI, the pinned binary download is preferable to `go install @latest` — plans embed the pgschema version and `apply` requires a matching version.

### Connection Options (shared by all commands)

| Flag | Env | Default | Notes |
|------|-----|---------|-------|
| `--host` | `PGHOST` | `localhost` | |
| `--port` | `PGPORT` | `5432` | |
| `--db` | `PGDATABASE` | — | required |
| `--user` | `PGUSER` | — | required |
| `--password` | `PGPASSWORD` | — | also resolvable via `.pgpass` |
| `--sslmode` | `PGSSLMODE` | `prefer` | `disable`/`allow`/`prefer`/`require`/`verify-ca`/`verify-full` |
| `--schema` | — | `public` | one schema per invocation |

Password precedence: `--password` flag → `PGPASSWORD` → `~/.pgpass` → interactive prompt. A `.env` file is also read (see [dotenv](https://www.pgschema.com/cli/dotenv)).

Note that pgschema takes **discrete connection flags**. A workflow wrapper should expose explicit host, port, database, user, and password values rather than relying on brittle URL parsing.

### `pgschema dump`

Extracts a schema in developer-friendly SQL (much cleaner than `pg_dump`).

```bash
PGPASSWORD=secret pgschema dump \
  --host localhost --db myapp --user postgres --schema public > schema.sql
```

| Flag | Default | Description |
|------|---------|-------------|
| `--multi-file` | `false` | Split output by object type into `tables/`, `views/`, `functions/`, ...; requires `--file` |
| `--file` | stdout | Output path; required with `--multi-file` |
| `--no-comments` | `false` | Drop per-object comment headers |
| `--qualify-schema` | `false` | Force `schema.object` qualification everywhere |

By default pgschema uses **smart qualification**: objects inside the dumped schema are unqualified, cross-schema references are qualified. This makes a dump reusable as a per-tenant template:

```bash
pgschema dump --schema template ... > template.sql
pgschema apply --schema tenant1 --file template.sql ...
pgschema apply --schema tenant2 --file template.sql ...
```

Multi-file output uses `\i` include directives in the main file, and `plan`/`apply` both resolve includes relative to the main file's directory.

### `pgschema plan`

```bash
PGPASSWORD=secret pgschema plan \
  --host localhost --db myapp --user postgres --schema public \
  --file schema.sql \
  --output-human plan.txt \
  --output-json plan.json \
  --output-sql migration.sql \
  --no-color
```

| Flag | Description |
|------|-------------|
| `--file` | Desired-state SQL file (required) |
| `--output-human` | `stdout` or a path — the reviewable summary |
| `--output-json` | `stdout` or a path — the artifact consumed by `apply --plan` |
| `--output-sql` | `stdout` or a path — raw DDL script |
| `--no-color` | Strip ANSI codes; use this in CI |

Only one output format may target `stdout`. With no flags at all, it prints the human format with colors.

Human output looks like:

```
Plan: 2 to add, 1 to modify.

Summary by type:
  tables: 1 to add, 1 to modify
  indexes: 1 to add

Tables:
  + public.posts
  ~ public.users
    + column name
    + column updated_at

Transaction: true

DDL to be executed:
--------------------------------------------------
ALTER TABLE users ADD COLUMN name VARCHAR(100);
...
```

The JSON output carries `pgschema_version`, `created_at`, `source_fingerprint.hash`, and a `diffs[]` array where each entry has `sql`, `type`, `operation`, `path`, and `can_run_in_transaction`.

How planning works internally: pgschema loads the desired-state SQL into a **temporary embedded PostgreSQL instance** to validate and normalize it, introspects the target schema, then diffs. Extensions and some cross-schema references need an [external plan database](https://www.pgschema.com/cli/plan-db) instead (`--plan-*` flags).

### `pgschema apply`

Two modes, mutually exclusive:

```bash
# Plan mode — recommended for CI/CD; validates the fingerprint
pgschema apply --host prod --db myapp --user deployer --plan plan.json --auto-approve

# File mode — plans internally, then applies; convenient for local dev
pgschema apply --host localhost --db myapp --user postgres --file schema.sql
```

| Flag | Default | Description |
|------|---------|-------------|
| `--plan` | — | Pre-generated plan JSON (mutually exclusive with `--file`) |
| `--file` | — | Desired-state SQL (mutually exclusive with `--plan`) |
| `--auto-approve` | `false` | Skip the interactive confirmation — required in CI |
| `--no-color` | `false` | Strip ANSI codes |
| `--lock-timeout` | unset | e.g. `30s`, `5m`; failed lock acquisition is retried with exponential backoff. `CREATE INDEX CONCURRENTLY` is never retried |
| `--application-name` | `pgschema` | Visible in `pg_stat_activity` |

Safety behaviors:

- **Fingerprint validation** (plan mode only) — apply aborts if the schema changed since the plan.
- **Version compatibility** — the pgschema version and plan format version must match.
- **Transaction handling** — changes run in transaction groups where possible; `CREATE INDEX CONCURRENTLY` and similar run outside. `VALIDATE CONSTRAINT` is always committed separately from its `ADD ... NOT VALID`.
- **No-op detection** — exits cleanly with "No changes to apply."

Setting `--lock-timeout` in production is strongly advised; without it PostgreSQL waits indefinitely for locks, which can stall a busy table behind a DDL statement.

### `.pgschemaignore`

Excludes objects from `dump`, `plan`, and `apply`. See [Ignore](https://www.pgschema.com/cli/ignore).

## Rollback

There is no `pgschema rollback`. Rollback is just a forward apply of an earlier schema file:

```bash
git checkout HEAD~1 schema.sql
pgschema plan  ... --file schema.sql --output-json rollback_plan.json --output-human rollback_plan.txt
cat rollback_plan.txt          # review — dropped columns mean data loss
pgschema apply ... --plan rollback_plan.json --lock-timeout 30s

# verify
pgschema dump ... > post_rollback_state.sql
diff schema.sql post_rollback_state.sql
```

Reverting a column addition means dropping the column and its data. Back up before rolling back.

## Environment Promotion

The upstream "local to production" pattern reuses **one** `plan.json` across environments:

```
dev DB ──dump──► schema.sql ──PR──► plan against staging ──► plan.json
                                                              │
                                             merge ───────────┤
                                                              ├─► apply to staging
                                                              └─► apply to production
```

Reusing the same plan for staging and production only works if the two schemas are actually identical — otherwise the production fingerprint check rejects the apply. That rejection is the feature: it catches drift from hotfixes and manual changes instead of silently applying an unreviewed diff.

A more conservative alternative is to plan per environment and accept that the production plan is reviewed at deploy time rather than at PR time.

## Reusable Workflows

| Workflow | Purpose |
|----------|---------|
| `pgschema-plan.yml` | Run `pgschema plan`, write human output to job summary, upload `plan.json`, `plan.txt`, and `migration.sql` |
| `pgschema-apply.yml` | Run `pgschema apply` from a reviewed plan artifact or directly from a schema file |
| `pgschema-dump.yml` | Run `pgschema dump`, upload SQL artifact, optionally commit `schema.sql` back to the caller repo |

All three workflows can use the shared `actions/setup-pgschema` composite action and optionally connect through WireGuard or Telepresence.

### Inputs

| Name | Type | Required | Default | Used by | Description |
|------|------|----------|---------|---------|-------------|
| `schema-file` | string | plan: yes, apply: conditional | `""` | plan, apply | Desired-state SQL file relative to repo root. In apply, mutually exclusive with `plan-artifact-name` |
| `plan-artifact-name` | string | no | `""` | apply | Artifact containing `plan.json` from `pgschema-plan.yml`; recommended for production |
| `plan-run-id` | string | no | `""` | apply | Workflow run ID to download `plan-artifact-name` from; empty means current run |
| `schema` | string | no | `public` | all | Target PostgreSQL schema name |
| `pgschema-version` | string | no | `v1.12.5` | all | Release tag for the downloaded binary; plan/apply must match |
| `artifact-name` | string | no | `pgschema-plan` / `pgschema-dump` | plan, dump | Uploaded artifact name |
| `artifact-retention-days` | number | no | `30` | plan, dump | Artifact retention period |
| `fail-on-changes` | boolean | no | `false` | plan | Fail when the generated plan contains changes |
| `lock-timeout` | string | no | `""` | apply | e.g. `30s` |
| `application-name` | string | no | `pgschema` | apply | PostgreSQL application name visible in `pg_stat_activity` |
| `environment` | string | no | `""` | apply | GitHub Environment approval gate |
| `multi-file` | boolean | no | `false` | dump | Emit a per-object-type directory layout |
| `no-comments` | boolean | no | `false` | dump | Omit per-object comment headers |
| `qualify-schema` | boolean | no | `false` | dump | Force `schema.object` qualification |
| `output-path` | string | no | `""` | dump | If set, commit the dump to this path |
| `target-branch` | string | no | `main` | dump | Branch to commit to when `output-path` is set |
| `commit-message` | string | no | `chore: update schema from pgschema dump` | dump | Commit message for dump commit-back |
| `also-proxy` | string | no | `""` | all | CIDR ranges to also proxy through Telepresence |

### Proposed secrets

| Name | Required | Description |
|------|----------|-------------|
| `db-host` | yes | Database host |
| `db-port` | no | Database port (default `5432`) |
| `db-name` | yes | Database name |
| `db-user` | yes | Database user |
| `db-password` | yes | Database password (exported as `PGPASSWORD`) |
| `wg-config-file` | no | WireGuard config for VPN tunnel |
| `kubeconfig` | no | Kubeconfig for Telepresence (mutually exclusive with `wg-config-file`) |
| `pat` | no | Token for committing dumps (required for dump with `output-path`) |

### Example: Plan on PR

```yaml
name: DB Schema Plan

on:
  pull_request:
    paths:
      - 'schema/**'

jobs:
  plan:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-plan.yml@main
    with:
      schema-file: schema/schema.sql
      schema: public
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
```

### Example: Manual Apply from Schema File

This is simpler but does not apply the exact reviewed `plan.json` from the PR.

```yaml
name: DB Schema Apply

on:
  workflow_dispatch:

jobs:
  apply:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-apply.yml@main
    with:
      schema-file: schema/schema.sql
      schema: public
      environment: production
      lock-timeout: 30s
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
```

### Example: Dump and Commit Bootstrap Schema

```yaml
name: DB Schema Bootstrap

on:
  workflow_dispatch:

jobs:
  dump:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-dump.yml@main
    with:
      schema: public
      output-path: schema/schema.sql
      target-branch: main
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
      pat: ${{ secrets.PAT }}
```

### Notes

- `pgschema-apply.yml` can apply from `plan-artifact-name` (recommended for reviewed plans) or from `schema-file` (simpler manual apply). These inputs are mutually exclusive.
- `pgschema-version` must match between plan and apply when using a plan artifact.
- Use `lock-timeout` for production apply jobs to avoid waiting indefinitely on PostgreSQL locks.
- Provide either `wg-config-file` or `kubeconfig`, never both.

## Reference Links

- [pgschema docs](https://www.pgschema.com/)
- [Quickstart](https://www.pgschema.com/quickstart)
- [Plan, Review, Apply](https://www.pgschema.com/workflow/plan-review-apply)
- [GitOps / CI examples](https://www.pgschema.com/workflow/gitops)
- [Example repo with working GitHub Actions](https://github.com/pgschema/github-actions-example)
