---
name: running-dbt-commands
description: Formats and executes dbt CLI commands via uv against Microsoft Fabric Spark lakehouses. Use when running models, tests, builds, compiles, or show queries via dbt CLI. Use when unsure how to format command parameters or which target to use (ephemeral_dev, ephemeral_dep, prod).
user-invocable: false
metadata:
  author: accelerate-data
---

# Running dbt Commands

## Execution Pattern

All dbt commands run through `uv` with the `.env` file for Fabric credentials:

```bash
uv run --env-file .env dbt <command> --target <target>
```

The `ephemeral_dev` target uses `vdstudio_oauth` authentication — the adapter fetches the OAuth token from VD Studio via `VD_STUDIO_TOKEN_URL` and `VD_STUDIO_USER_ID` (both set in `.env`). No manual token refresh is needed.

## Targets

| Target | When to use |
|--------|-------------|
| `ephemeral_dev` | **Default.** Local development iteration — fast, sub-minute feedback via Livy |
| `ephemeral_dep` | Deployment validation (runs inside Fabric notebook) |
| `prod` | Production scheduled runs (runs inside Fabric notebook) |

If the user does not specify a target, always use `--target ephemeral_dev`.

## Preferences

1. **Use `build` instead of `run` or `test`** — `test` doesn't refresh the model, so testing a model change requires `build`. `build` does a `run` and a `test` of each node (model, seed, snapshot) in the order of the DAG
2. **Always use `--quiet`** with `--warn-error-options '{"error": ["NoNodesForSelectionCriteria"]}'` to reduce output while catching selector typos
3. **Always use `--select`** — never run the entire project without explicit user approval

## Quick Reference

```bash
# Standard command pattern (default target: ephemeral_dev)
uv run --env-file .env dbt build --select my_model --target ephemeral_dev --quiet --warn-error-options '{"error": ["NoNodesForSelectionCriteria"]}'

# Run all models
uv run --env-file .env dbt run --target ephemeral_dev

# Preview model output
uv run --env-file .env dbt show --select my_model --target ephemeral_dev --limit 10

# Run inline SQL query
uv run --env-file .env dbt show --inline "select * from {{ ref('orders') }}" --target ephemeral_dev --limit 5

# With variables (JSON format for multiple)
uv run --env-file .env dbt build --select my_model --target ephemeral_dev --vars '{"key": "value"}'

# Full refresh for incremental models
uv run --env-file .env dbt build --select my_model --target ephemeral_dev --full-refresh

# List resources before running
uv run --env-file .env dbt list --select my_model+ --target ephemeral_dev --resource-type model

# Deployment validation
uv run --env-file .env dbt build --select my_model --target ephemeral_dep --quiet

# Production run
uv run --env-file .env dbt run --target prod
```

## dbt CLI — uv Only

This project uses **dbt Core** via `uv` with the `vd-dbt-fabricspark` adapter. Do not use `dbtf`, dbt Fusion, or dbt Cloud CLI.

```bash
# Verify installation
uv pip show dbt-core
uv pip show vd-dbt-fabricspark
```

## Selectors

**Always provide a selector.** Graph operators:

| Operator | Meaning | Example |
|----------|---------|---------|
| `model+` | Model and all downstream | `stg_orders+` |
| `+model` | Model and all upstream | `+dim_customers` |
| `+model+` | Both directions | `+orders+` |
| `model+N` | Model and N levels downstream | `stg_orders+1` |

```bash
--select my_model              # Single model
--select staging.*             # Path pattern
--select fqn:*stg_*            # FQN pattern
--select model_a model_b       # Union (space)
--select tag:x,config.mat:y    # Intersection (comma)
--exclude my_model             # Exclude from selection
```

**Resource type filter:**
```bash
--resource-type model
--resource-type test --resource-type unit_test
```

Valid types: `model`, `test`, `unit_test`, `snapshot`, `seed`, `source`, `exposure`, `metric`, `semantic_model`, `saved_query`, `analysis`

## List

Use `dbt list` to preview what will be selected before running. Helpful for validating complex selectors.

```bash
dbt list --select my_model+              # Preview selection
dbt list --select my_model+ --resource-type model  # Only models
dbt list --output json                   # JSON output
dbt list --select my_model --output json --output-keys unique_id name resource_type config
```

**Available output keys for `--output json`:**
`unique_id`, `name`, `resource_type`, `package_name`, `original_file_path`, `path`, `alias`, `description`, `columns`, `meta`, `tags`, `config`, `depends_on`, `patch_path`, `schema`, `database`, `relation_name`, `raw_code`, `compiled_code`, `language`, `docs`, `group`, `access`, `version`, `fqn`, `refs`, `sources`, `metrics`

## Show

Preview data with `dbt show`. Use `--inline` for arbitrary SQL queries.

```bash
dbt show --select my_model --limit 10
dbt show --inline "select * from {{ ref('orders') }} where status = 'pending'" --limit 5
```

**Important:** Use `--limit` flag, not SQL `LIMIT` clause.

## Variables

Pass as STRING, not dict. No special characters (`\`, `\n`).

```bash
--vars 'my_var: value'                              # Single
--vars '{"k1": "v1", "k2": 42, "k3": true}'         # Multiple (JSON)
```

## Analyzing Run Results

After a dbt command, check `target/run_results.json` for detailed execution info:

```bash
# Quick status check
cat target/run_results.json | jq '.results[] | {node: .unique_id, status: .status, time: .execution_time}'

# Find failures
cat target/run_results.json | jq '.results[] | select(.status != "success")'
```

**Key fields:**
- `status`: success, error, fail, skipped, warn
- `execution_time`: seconds spent executing
- `compiled_code`: rendered SQL
- `adapter_response`: database metadata (rows affected, bytes processed)

## Defer (Skip Upstream Builds)

Reference production data instead of building upstream models:

```bash
dbt build --select my_model --defer --state prod-artifacts
```

**Flags:**
- `--defer` - enable deferral to state manifest
- `--state <path>` - path to manifest from previous run (e.g., production artifacts)
- `--favor-state` - prefer node definitions from state even if they exist locally

```bash
dbt build --select my_model --defer --state prod-artifacts --favor-state
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Running `dbt` directly without `uv run --env-file .env` | Always use `uv run --env-file .env dbt ...` — env vars are required |
| Forgetting `--target ephemeral_dev` | Always specify the target explicitly |
| Using `test` after model change | Use `build` — test doesn't refresh the model |
| Running without `--select` | Always specify what to run |
| Using `--quiet` without warn-error | Add `--warn-error-options '{"error": ["NoNodesForSelectionCriteria"]}'` |
| Adding LIMIT to SQL in `dbt show` | Use `--limit` flag instead |
| Vars with special characters | Pass as simple string, no `\` or `\n` |
| OAuth token errors (401) | Check `VD_STUDIO_TOKEN_URL` and `VD_STUDIO_USER_ID` are set in `.env` — adapter fetches token automatically |
| Livy session timeout | Sessions are reused via `/tmp/livy-session-id.txt` — delete the file to force a new session |
