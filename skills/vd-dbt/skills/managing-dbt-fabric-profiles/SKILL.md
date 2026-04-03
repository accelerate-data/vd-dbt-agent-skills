---
name: managing-dbt-fabric-profiles
description: Manages dbt profiles.yml for Microsoft Fabric Spark (vd-dbt-fabricspark adapter). Use when setting up, verifying, or modifying profiles.yml targets (ephemeral_dev, ephemeral_dep, prod) for local or deployed dbt execution against Fabric lakehouses.
allowed-tools: "Bash(cat *), Bash(grep *), Read, Write, Edit, Glob"
user-invocable: false
metadata:
  author: accelerate-data
---

# Managing dbt Fabric Profiles

Set up and maintain `profiles.yml` for the `vd-dbt-fabricspark` adapter with three execution targets.

## Three-Target Architecture

| Target | Auth Method | Where It Runs | Purpose |
|--------|-------------|---------------|---------|
| `ephemeral_dev` | `vdstudio_oauth` | Local machine (via `uv run`) | Design-phase iteration — fast, sub-minute feedback |
| `ephemeral_dep` | `fabric_notebook` | Fabric notebook | Deployment validation |
| `prod` | `fabric_notebook` | Fabric notebook | Production scheduled runs |

All targets share the same Livy/Spark configuration. Differences:

- `ephemeral_dev` uses `vdstudio_oauth` auth — reads `VD_STUDIO_TOKEN_URL` + `VD_STUDIO_USER_ID` from `.env`. Set automatically on intent creation. `VD_STUDIO_USER_ID` is updated on each user login (single-user per intent).
- `ephemeral_dep` uses `fabric_notebook` auth — runs inside Fabric notebook runtime. Points to ephemeral workspace (dynamic via `env_var()`).
- `prod` uses `fabric_notebook` auth — runs inside Fabric notebook runtime. Points to **domain workspace with hardcoded values** (not `env_var()`). Read `DOMAIN_*` variables from `.env` and write them as literal strings into `profiles.yml`.

**Default target is `ephemeral_dev`** — local iteration is the primary use case.

## Required Environment Variables

All targets use `env_var()` for dynamic values. These must be present in the clone's `.env` file:

### Set on intent creation (static per intent)

| Variable | Source | Example |
|----------|--------|---------|
| `WORKSPACE_ID` | Domain's Fabric workspace config | `cb696f8e-9b93-49bf-b8dd-b7ff5ea60aca` |
| `WORKSPACE_NAME` | Domain's Fabric workspace config | `ephm_yo__new-intent-4dfbd045` |
| `LAKEHOUSE_ID` | Domain's Fabric lakehouse config | `f9a1fab1-370a-4ac9-9456-3b84493332c9` |
| `LAKEHOUSE` | Domain's Fabric lakehouse name | `ai_lake2_new_intent_4dfbd045` |
| `SCHEMA` | Default: `dbo` | `dbo` |
| `DBT_JOB_NAME` | Default: `dbt_spark_job` | `dbt_spark_job` |
| `VD_STUDIO_TOKEN_URL` | VD Studio token endpoint | `http://127.0.0.1:3238/az_token` |
| `DOMAIN_WORKSPACE_ID` | Domain workspace GUID | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` |
| `DOMAIN_WORKSPACE_NAME` | Domain workspace name | `sample_wp2` |
| `DOMAIN_LAKEHOUSE_ID` | Domain lakehouse GUID | `12eff595-0528-4a26-923b-7676352f4423` |
| `DOMAIN_LAKEHOUSE` | Domain lakehouse name | `ai_lake2` |
| `DOMAIN_SCHEMA` | Domain schema | `dbo` |
| `DOMAIN_LAKEHOUSE_SQL_ENDPOINT` | Domain SQL endpoint | `xyz.datawarehouse.fabric.microsoft.com` |

### Updated on each user login

| Variable | Source | Example |
|----------|--------|---------|
| `VD_STUDIO_USER_ID` | Logged-in user's ID | `383b3e33-2d96-461d-a409-853c4561c53d` |

> **Note:** Only one user is supported per intent. `VD_STUDIO_USER_ID` is overwritten each time a different user logs in.

## Full profiles.yml Template

When `profiles.yml` is missing or incomplete, generate this complete profile:

```yaml
dbt_fab_spark:
  outputs:

    ephemeral_dev:
      authentication: vdstudio_oauth
      endpoint: https://msitapi.fabric.microsoft.com/v1
      workspaceid: "{{ env_var('WORKSPACE_ID') }}"
      workspace_name: "{{ env_var('WORKSPACE_NAME') }}"
      lakehouse: "{{ env_var('LAKEHOUSE') }}"
      lakehouseid: "{{ env_var('LAKEHOUSE_ID') }}"
      lakehouse_schemas_enabled: true
      schema: "{{ env_var('SCHEMA') }}"
      method: livy
      type: fabricspark
      reuse_livy_session: true
      session_id_file: ./.livy-session-id.txt
      spark_config:
        name: "{{ env_var('DBT_JOB_NAME') }}"
        driverCores: 4
        driverMemory: 28g
        executorCores: 4
        executorMemory: 28g
      connect_retries: 2
      connect_timeout: 500
      threads: 10

    ephemeral_dep:
      authentication: fabric_notebook
      endpoint: https://msitapi.fabric.microsoft.com/v1
      workspaceid: "{{ env_var('WORKSPACE_ID') }}"
      workspace_name: "{{ env_var('WORKSPACE_NAME') }}"
      lakehouse: "{{ env_var('LAKEHOUSE') }}"
      lakehouseid: "{{ env_var('LAKEHOUSE_ID') }}"
      lakehouse_schemas_enabled: true
      schema: "{{ env_var('SCHEMA') }}"
      method: livy
      type: fabricspark
      reuse_livy_session: true
      session_id_file: ./.livy-session-id.txt
      spark_config:
        name: "{{ env_var('DBT_JOB_NAME') }}"
        driverCores: 4
        driverMemory: 28g
        executorCores: 4
        executorMemory: 28g
      connect_retries: 2
      connect_timeout: 500
      threads: 10

    # prod target uses HARDCODED domain values — read DOMAIN_* from .env
    # and write them as literal strings here (not env_var)
    prod:
      authentication: fabric_notebook
      endpoint: https://msitapi.fabric.microsoft.com/v1
      workspaceid: "<DOMAIN_WORKSPACE_ID>"
      workspace_name: "<DOMAIN_WORKSPACE_NAME>"
      lakehouse: "<DOMAIN_LAKEHOUSE>"
      lakehouseid: "<DOMAIN_LAKEHOUSE_ID>"
      lakehouse_schemas_enabled: true
      schema: "<DOMAIN_SCHEMA>"
      method: livy
      type: fabricspark
      reuse_livy_session: true
      session_id_file: ./.livy-session-id.txt
      spark_config:
        name: dbt_spark_job
        driverCores: 4
        driverMemory: 28g
        executorCores: 4
        executorMemory: 28g
      connect_retries: 2
      connect_timeout: 500
      threads: 10

  target: ephemeral_dev
```

## Profile Name

The profile name in `profiles.yml` must match the `profile:` field in `dbt_project.yml`. The standard profile name is `dbt_fab_spark`.

## Adding Missing Targets

If any target is missing, add it under `dbt_fab_spark.outputs`. Use the same Spark/Livy config block with these differences:

- `ephemeral_dev` → `authentication: vdstudio_oauth`, dynamic `env_var()` values (ephemeral workspace)
- `ephemeral_dep` → `authentication: fabric_notebook`, dynamic `env_var()` values (ephemeral workspace)
- `prod` → `authentication: fabric_notebook`, **hardcoded** domain values (read `DOMAIN_*` from `.env`, write as literal strings)

## Common Issues

| Issue | Fix |
|-------|-----|
| Missing `ephemeral_dev` target | Add with `authentication: vdstudio_oauth` |
| Missing `ephemeral_dep` target | Add with `authentication: fabric_notebook` |
| `ephemeral_dev` uses `fabric_notebook` auth | Change to `vdstudio_oauth` — local execution needs VD Studio OAuth |
| `.env` missing `VD_STUDIO_TOKEN_URL` | Set on intent creation — check intent setup |
| `.env` missing `VD_STUDIO_USER_ID` | Updated on user login — check auth flow |
| Profile name mismatch | Ensure `profiles.yml` top-level key matches `dbt_project.yml` `profile:` |
| `target: dev` instead of `target: ephemeral_dev` | Update default target to `ephemeral_dev` |

## Spark Config Tuning

The default Spark config works for most workloads. Adjust only when:

| Parameter | Default | When to change |
|-----------|---------|----------------|
| `driverCores` | 4 | Large orchestration jobs |
| `driverMemory` | 28g | Memory pressure on driver |
| `executorCores` | 4 | Parallelism tuning |
| `executorMemory` | 28g | Large data shuffles |
| `threads` | 10 | More/fewer concurrent models |
| `connect_retries` | 2 | Flaky network to Fabric |
| `connect_timeout` | 500 | Slow Fabric API responses |
