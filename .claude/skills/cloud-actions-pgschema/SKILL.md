---
name: cloud-actions-pgschema
description: Generate GitHub Actions caller workflows for PostgreSQL schema management with pgschema (plan, apply & dump). Use this skill whenever the user mentions pgschema, PostgreSQL schema migrations, declarative schema plan/apply, schema diff for Postgres, database CI/CD for Postgres, or wants to set up automated PostgreSQL schema management in GitHub Actions. Also use when the user asks about cloud-actions pgschema workflows, DB schema pipelines for Postgres, or database schema bootstrapping/dumping for Postgres.
---

# pgschema Schema Workflows

Generate caller workflow YAML for the `NFUChen/cloud-actions` pgschema reusable workflows. These workflows manage PostgreSQL schema changes using [pgschema](https://www.pgschema.com/) declarative schema migration -- a Terraform-style plan/apply tool that supports PostgreSQL only.

See `docs/PGSCHEMA.md` in this repo for full CLI background, safety mechanisms (fingerprinting), and rollback guidance.

## Available Workflows

| Workflow | File | Purpose |
|----------|------|---------|
| Schema Plan | `pgschema-plan.yml` | Generate a migration plan, write human output to job summary, upload `plan.json`/`plan.txt`/`migration.sql` as an artifact |
| Schema Apply | `pgschema-apply.yml` | Apply schema changes -- either from a reviewed plan artifact (fingerprint-validated) or directly from a schema file |
| Schema Dump | `pgschema-dump.yml` | Dump the live database schema, upload as an artifact, optionally commit `schema.sql` back to the repo |

There is also a composite action, `actions/setup-pgschema`, which all three workflows use internally to install the pgschema CLI. Callers normally don't need to reference it directly unless building a custom job.

## When to Generate What

- **PR-triggered plan**: User wants to preview DB changes when a schema file changes in a pull request
- **Manual apply from schema file**: Simple case -- user wants to apply changes directly without a separate reviewed plan artifact
- **Plan + apply pair with plan artifact**: More rigorous -- plan on PR, then apply consumes the exact reviewed `plan.json` and pgschema validates the schema fingerprint hasn't drifted since
- **Dump (audit)**: User wants to see or export the current live database schema on demand
- **Dump (bootstrap)**: User wants to extract the current DB schema and commit it as `schema.sql` to the repo

## Caller Workflow Reference

### Base URL

```
NFUChen/cloud-actions/.github/workflows/<workflow-file>@main
```

### Connection secrets (shared by all three workflows)

These workflows take **discrete connection secrets**, not a single `db-url`. This matches the pgschema CLI, which takes `--host`/`--port`/`--db`/`--user`/`--password` flags rather than a URL.

| Name | Required | Description |
|------|----------|-------------|
| `db-host` | yes | Database server host |
| `db-port` | no | Database server port (defaults to `5432` when empty) |
| `db-name` | yes | Database name |
| `db-user` | yes | Database user name |
| `db-password` | yes | Database password (exported as `PGPASSWORD`) |
| `db-sslmode` | no | SSL mode (`disable`/`allow`/`prefer`/`require`/`verify-ca`/`verify-full`); defaults to `prefer` |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel |
| `kubeconfig` | no | Kubeconfig file content (base64 encoded) for Telepresence connection to K8s cluster (mutually exclusive with `wg-config-file`) |

### Schema Plan -- Inputs

```yaml
jobs:
  plan:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-plan.yml@main
    with:
      schema-file: "schema/schema.sql"      # required -- desired-state SQL file
      schema: "public"                       # optional -- default: public
      pgschema-version: "v1.12.5"           # optional -- default: v1.12.5; must match apply
      artifact-name: "pgschema-plan"         # optional -- default: pgschema-plan
      artifact-retention-days: 30            # optional -- default: 30
      fail-on-changes: false                 # optional -- default: false; fail the job if the plan has any diffs (drift detection)
      also-proxy: ""                         # optional -- CIDR ranges for Telepresence
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
```

### Schema Apply -- Inputs

Two mutually exclusive modes -- set exactly one of `plan-artifact-name` or `schema-file`.

```yaml
jobs:
  apply:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-apply.yml@main
    with:
      # Mode 1: apply a reviewed plan artifact (recommended for production)
      plan-artifact-name: "pgschema-plan"    # optional -- artifact name from pgschema-plan.yml
      plan-run-id: ""                        # optional -- run ID to fetch the artifact from; empty = current run

      # Mode 2: apply directly from a schema file (simpler, no separate review step)
      schema-file: ""                        # optional -- mutually exclusive with plan-artifact-name

      schema: "public"                       # optional -- default: public
      pgschema-version: "v1.12.5"           # optional -- must match the version that produced the plan
      lock-timeout: ""                       # optional -- e.g. "30s"; strongly recommended for production
      application-name: "pgschema"          # optional -- visible in pg_stat_activity
      environment: ""                        # optional -- GitHub Environment name for an approval gate
      also-proxy: ""                         # optional -- CIDR ranges for Telepresence
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
```

### Schema Dump -- Inputs

```yaml
jobs:
  dump:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-dump.yml@main
    with:
      schema: "public"                       # optional -- default: public
      pgschema-version: "v1.12.5"           # optional -- default: v1.12.5
      multi-file: false                      # optional -- split output by object type
      no-comments: false                     # optional -- omit per-object comment headers
      qualify-schema: false                  # optional -- force schema.object qualification everywhere
      output-path: ""                        # optional -- commit dump to this path (requires pat secret)
      target-branch: "main"                 # optional -- branch to commit to when output-path is set
      commit-message: "chore: update schema from pgschema dump"  # optional
      artifact-name: "pgschema-dump"         # optional -- default: pgschema-dump
      artifact-retention-days: 30            # optional -- default: 30
      also-proxy: ""                         # optional -- CIDR ranges for Telepresence
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
      pat: ${{ secrets.PAT }}                # required only when output-path is set
```

### Inputs Detail

| Name | Type | Required | Default | Used by | Description |
|------|------|----------|---------|---------|-------------|
| `schema-file` | string | plan: yes; apply: conditional | -- | plan, apply | Desired-state SQL file relative to repo root. In apply, mutually exclusive with `plan-artifact-name` |
| `plan-artifact-name` | string | no | `""` | apply | Artifact from `pgschema-plan.yml` containing the reviewed `plan.json` |
| `plan-run-id` | string | no | `""` | apply | Workflow run ID to download `plan-artifact-name` from; empty means current run |
| `schema` | string | no | `public` | all | Target PostgreSQL schema name |
| `pgschema-version` | string | no | `v1.12.5` | all | Release tag for the downloaded pgschema binary |
| `artifact-name` | string | no | `pgschema-plan` / `pgschema-dump` | plan, dump | Uploaded artifact name |
| `artifact-retention-days` | number | no | `30` | plan, dump | Artifact retention period in days |
| `fail-on-changes` | boolean | no | `false` | plan | Fail the job when the plan contains any diffs |
| `lock-timeout` | string | no | `""` | apply | e.g. `30s`, `5m`; empty waits on locks indefinitely |
| `application-name` | string | no | `pgschema` | apply | PostgreSQL `application_name`, visible in `pg_stat_activity` |
| `environment` | string | no | `""` | apply | GitHub Environment name for a manual approval gate |
| `multi-file` | boolean | no | `false` | dump | Emit a per-object-type directory layout with `\i` includes |
| `no-comments` | boolean | no | `false` | dump | Omit per-object comment headers |
| `qualify-schema` | boolean | no | `false` | dump | Force `schema.object` qualification everywhere |
| `output-path` | string | no | `""` | dump | If set, commits the dump to this path in the repo |
| `target-branch` | string | no | `main` | dump | Branch to commit to when `output-path` is set |
| `commit-message` | string | no | `chore: update schema from pgschema dump` | dump | Commit message used for the commit-back |
| `also-proxy` | string | no | `""` | all | Comma-separated CIDR ranges to also proxy through Telepresence |

### Secrets Detail

| Name | Required | Description |
|------|----------|-------------|
| `db-host` | yes | Database server host |
| `db-port` | no | Database server port (defaults to `5432`) |
| `db-name` | yes | Database name |
| `db-user` | yes | Database user name |
| `db-password` | yes | Database password (exported as `PGPASSWORD`) |
| `db-sslmode` | no | SSL mode; defaults to `prefer` |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel |
| `kubeconfig` | no | Kubeconfig file content (base64 encoded) for Telepresence connection to K8s cluster (mutually exclusive with `wg-config-file`) |
| `pat` | no | Personal access token for pushing commits (required for dump with `output-path`) |

## Complete Example: Plan on PR + Manual Apply from Schema File

```yaml
# .github/workflows/db-plan.yml
name: DB Schema Plan

on:
  pull_request:
    paths:
      - 'schema/**'

jobs:
  plan:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-plan.yml@main
    with:
      schema-file: "schema/schema.sql"
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
```

```yaml
# .github/workflows/db-apply.yml
name: DB Schema Apply

on:
  workflow_dispatch:

jobs:
  apply:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-apply.yml@main
    with:
      schema-file: "schema/schema.sql"
      environment: production
      lock-timeout: "30s"
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
```

## Complete Example: Plan + Apply Sharing a Reviewed Plan Artifact

Use this when the caller wants apply to execute the *exact* diff that was reviewed on the PR, with pgschema's fingerprint check guarding against concurrent drift. Both jobs must pin the same `pgschema-version`.

```yaml
name: DB Schema Deploy

on:
  push:
    branches: [main]
    paths:
      - 'schema/**'

jobs:
  plan:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-plan.yml@main
    with:
      schema-file: "schema/schema.sql"
      pgschema-version: "v1.12.5"
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}

  apply:
    needs: plan
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-apply.yml@main
    with:
      plan-artifact-name: "pgschema-plan"
      pgschema-version: "v1.12.5"
      environment: production
      lock-timeout: "30s"
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
```

Note: `plan-run-id` is only needed when apply runs in a *different* workflow run than plan (e.g. apply triggered later via `workflow_dispatch` against a PR's plan). When both jobs are in the same workflow run, as above, the default (current run) is correct and `plan-run-id` can be omitted.

## Complete Example: Dump (Audit)

```yaml
# .github/workflows/db-inspect.yml
name: DB Schema Inspect

on:
  workflow_dispatch:

jobs:
  dump:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-dump.yml@main
    with:
      schema: "public"
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
```

## Complete Example: Dump (Bootstrap, Commit Back)

```yaml
# .github/workflows/db-bootstrap.yml
name: DB Schema Bootstrap

on:
  workflow_dispatch:

jobs:
  bootstrap:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-dump.yml@main
    with:
      schema: "public"
      output-path: "schema/schema.sql"
      target-branch: "main"
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-port: ${{ secrets.DB_PORT }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
      pat: ${{ secrets.PAT }}
```

## Complete Example: Plan via Telepresence

```yaml
# .github/workflows/db-plan-k8s.yml
name: DB Schema Plan (K8s)

on:
  pull_request:
    paths:
      - 'schema/**'

jobs:
  plan:
    uses: NFUChen/cloud-actions/.github/workflows/pgschema-plan.yml@main
    with:
      schema-file: "schema/schema.sql"
      also-proxy: "10.0.0.0/8"
    secrets:
      db-host: ${{ secrets.DB_HOST }}
      db-name: ${{ secrets.DB_NAME }}
      db-user: ${{ secrets.DB_USER }}
      db-password: ${{ secrets.DB_PASSWORD }}
      kubeconfig: ${{ secrets.KUBECONFIG }}
```

## Prerequisites for the Caller Repo

1. A pgschema-compatible SQL schema file in the repo (e.g. `schema/schema.sql`) -- required for plan/apply, not needed for dump
2. `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` secrets configured (plus `DB_PORT`/`DB_SSLMODE` if non-default)
3. (Optional) `WG_CONFIG_FILE` secret if the database is behind a VPN
4. (Optional) `KUBECONFIG` secret if the database is inside a K8s cluster (Telepresence; mutually exclusive with WireGuard)
5. (Optional) `PAT` secret with repo write permissions -- required for dump with `output-path`

## Generation Guidelines

When generating caller workflows:

- Always ask the user for their schema file path if not provided (for plan/apply)
- pgschema only supports PostgreSQL -- if the user mentions MySQL, MariaDB, or SQLite, this skill does not apply; point them to a database-agnostic tool instead
- Ask whether the caller wants a **reviewed-plan handoff** (plan uploads an artifact, apply consumes it with `plan-artifact-name`) or a **simpler direct apply** (`schema-file` on both, no shared artifact). Default to the reviewed-plan pattern for production deployments, and direct `schema-file` apply for lower environments or quick iteration.
- When using the plan-artifact handoff, both `pgschema-plan.yml` and `pgschema-apply.yml` calls must set the same `pgschema-version` -- `apply --plan` refuses to run against a plan produced by a different version
- Use discrete `db-*` secrets, never a combined `db-url` -- pgschema's CLI takes discrete connection flags
- Include `wg-config-file` secret only if the user mentions VPN or private network access
- Include `kubeconfig` secret if the user mentions Kubernetes, cluster-internal, or Telepresence
- WireGuard and Telepresence are mutually exclusive -- never include both in the same caller workflow
- If using Telepresence, ask if the user needs `also-proxy` for additional CIDR ranges
- Use `paths` filter on PR trigger to only run plan when schema files change
- Use `workflow_dispatch` or a merge-triggered job for apply so it doesn't run on every PR push
- Recommend `lock-timeout` (e.g. `30s`) for production apply jobs to avoid indefinitely blocking on busy tables
- Recommend `environment` on the apply job when deploying to production, to get a manual approval gate
- The `fail-on-changes` input on plan is a drift-detection pattern -- only suggest it for scheduled jobs that assert the live DB matches the schema file, not for normal PR plans
- For dump: ask the user which schema to dump if not specified (default `public`)
- For dump with commit-back: require `output-path`, and remind the user `PAT` secret is mandatory in that case
- The `pgschema-version` input rarely needs to change -- only override if the user needs a specific pgschema release
