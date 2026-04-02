---
name: fabric-dbt-ephemeral
description: Runs dbt inside an ephemeral Fabric workspace safely against production data using --defer, dbt clone, and a custom source() macro. Use when developing, fixing, or testing dbt models in an ephemeral workspace that reads from a domain (production) lakehouse.
allowed-tools: "Bash(uv run *), Bash(cat *), Bash(grep *), Read, Write, Edit, Glob"
user-invocable: false
metadata:
  author: accelerate-data
---

# Ephemeral dbt Runner

You are operating inside an **ephemeral workspace** — a short-lived isolated Fabric workspace provisioned for a single user intent. You have your own empty lakehouse. The **domain workspace** holds all production data (Bronze, Silver, Gold).

Goal: develop, fix, and test dbt models safely — reading from production where needed, writing into the ephemeral lakehouse — without ever touching the domain workspace. When done, the user raises a PR on the dbt project only.

## Core Constraints

- **Never make the dbt project ephemeral-specific.** Every change must work identically when run against domain. No `{% if target.name == 'ephemeral_dev' %}` conditions in model SQL. No hardcoded ephemeral coordinates anywhere.
- **Domain lakehouse is read-only.** Never run dbt against it. Never write to it.
- **The dbt project is the source of truth.** The ephemeral workspace is a sandbox only.

## Key Concepts

### --defer

Enables sandbox behaviour. For every `ref()` call, dbt checks if the table exists in the ephemeral lakehouse first. If yes → reads from ephemeral. If no → falls back to domain using prod manifest coordinates. Requires a `manifest.json` from the last successful domain run passed via `--state`.

### dbt clone

Creates zero-copy shallow clones of domain tables into ephemeral. Copies only metadata — no data moved.

Clone is only relevant for **incremental models**. Table models always do `CREATE OR REPLACE` — cloning them is pointless, skip them.

Clone an incremental model when any of these are true:
1. **SQL changed** — incremental logic modified vs prod manifest → `--select state:modified+`
2. **Source data changed** — dlt ingested new data and downstream includes this incremental → `--select +<model>`
3. **Node selected via `+`** — upstream incremental gets selected by the command → `--select +<model>`

### source() behaviour

dbt's native `--defer` has no effect on `source()`. A custom macro is required to give sources the same defer-aware behaviour as `ref()`.

## First-Time Setup — Source Macro

Before running any dbt commands, check whether `macros/vd-studio/source.sql` exists. If not, create it:

```sql
{% macro source(source_name, table_name) %}

  {% set original_rel = builtins.source(source_name, table_name) %}

  {% if execute %}
    {% set original_str = original_rel | string %}

    {# Fabric Spark: target.database is empty. Use target.get('lakehouse') instead #}
    {% set target_lakehouse = target.get('lakehouse', '') if target.get is defined else '' %}

    {# Check if source points to a different lakehouse than our current dev target #}
    {% if target_lakehouse and target_lakehouse not in original_str %}
      {# Source is cross-env. Check if table exists in dev lakehouse #}
      {% set dev_rel = adapter.get_relation(
          database=none,
          schema=target.schema,
          identifier=original_rel.identifier
      ) %}
      {% if dev_rel is not none %}
        {% do return(dev_rel) %}
      {% else %}
      {% endif %}
    {% endif %}
  {% endif %}

  {% do return(original_rel) %}

{% endmacro %}
```

This makes `source()` prefer ephemeral if the table exists there (e.g. ingested by dlt), falls back to domain if not. Commit this file as part of the first PR if it was not already present.

## Commands

All commands use `uv run --env-file .env dbt ...` and target `ephemeral_dev`.

### Standard run — SQL changed vs prod

```bash
uv run --env-file .env dbt clone --select state:modified+ --state prod-manifest --target ephemeral_dev
uv run --env-file .env dbt run --select state:modified+ --defer --state prod-manifest --target ephemeral_dev
```

### Source data updated via dlt — run downstream

```bash
uv run --env-file .env dbt clone --select +<model> --state prod-manifest --target ephemeral_dev
uv run --env-file .env dbt run --select +<model> --defer --state prod-manifest --target ephemeral_dev
```

### User explicitly selects upstream via `+`

```bash
uv run --env-file .env dbt clone --select +<model> --state prod-manifest --target ephemeral_dev
uv run --env-file .env dbt run --select +<model> --defer --state prod-manifest --target ephemeral_dev
```

Clone only applies to incremental models in the selection — table models are unaffected. Clone is a no-op for models already in ephemeral.

### Test

```bash
uv run --env-file .env dbt test --select <selection> --defer --state prod-manifest --target ephemeral_dev
```

### Build (run + test together)

```bash
uv run --env-file .env dbt clone --select <selection> --state prod-manifest --target ephemeral_dev
uv run --env-file .env dbt build --select <selection> --defer --state prod-manifest --target ephemeral_dev
```

### Seeds

```bash
uv run --env-file .env dbt seed --target ephemeral_dev
uv run --env-file .env dbt run --select <selection> --defer --state prod-manifest --target ephemeral_dev
```

Always run `dbt seed` explicitly before models that depend on seeds. Modified CSVs are never auto-uploaded by `dbt run`.

### Snapshots

```bash
uv run --env-file .env dbt snapshot --target ephemeral_dev --defer --state prod-manifest
```

If a snapshot has a hardcoded `target_schema` string, flag it to the user. It must use `target.schema ~ '_snapshots'` for proper isolation.

### Fresh project (no prod manifest yet)

**Domain lakehouse is empty — brand new project:**
```bash
uv run --env-file .env dbt run --target ephemeral_dev
```
No `--defer`, no `--state`, no clone. Plain run.

**Domain has data but no manifest generated yet:**
```bash
uv run --env-file .env dbt run --target ephemeral_dev
```
No `--defer` since there is no manifest. If dlt has ingested sources into ephemeral, the custom `source()` macro resolves them correctly without defer. Warn the user that upstream domain models are not accessible via defer until a manifest is generated from a successful prod run.

## What You Always Do

1. Check for `macros/vd-studio/source.sql` — create it if missing before any dbt command
2. Confirm prod manifest is available before using `--defer`
3. Apply correct clone decision before `dbt run` — see clone section for the three triggers
4. Always use `--target ephemeral_dev`

## What You Never Do

- Run any dbt command with `--target prod` or any target that writes to domain
- Hardcode ephemeral workspace names, lakehouse IDs, or workspace-specific paths into the dbt project
- Add environment-specific conditions into model SQL or sources.yml
- Skip `dbt clone` for incremental models — silent full reloads are expensive and produce incorrect incremental behaviour
