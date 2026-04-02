---
name: managing-dbt-fabric-profiles
description: Manages dbt profiles.yml for Microsoft Fabric Spark (vd-dbt-fabricspark adapter). Use when setting up or modifying profiles.yml targets (ephemeral_dev, ephemeral_dep, prod) for local or deployed dbt execution against Fabric lakehouses.
allowed-tools: "Bash(cat *), Bash(grep *), Read, Write, Edit, Glob"
user-invocable: false
metadata:
  author: accelerate-data
---

# Managing dbt Fabric Profiles

## Three-Target Architecture

| Target | Auth | Purpose |
|--------|------|---------|
| `ephemeral_dev` | `vdstudio_oauth` | Local dev via `uv run` — reads `VD_STUDIO_TOKEN_URL` + `VD_STUDIO_USER_ID` from `.env` |
| `ephemeral_dep` | `fabric_notebook` | Deployment validation inside Fabric notebook |
| `prod` | `fabric_notebook` | Production scheduled runs inside Fabric notebook |

Default target: `ephemeral_dev`. Only difference between targets is `authentication`.

## Environment Variables (`.env`)

Set on intent creation: `WORKSPACE_ID`, `WORKSPACE_NAME`, `LAKEHOUSE_ID`, `LAKEHOUSE`, `SCHEMA`, `DBT_JOB_NAME`, `VD_STUDIO_TOKEN_URL`

Updated on each user login: `VD_STUDIO_USER_ID` (single-user per intent, overwritten on login)

## Full profiles.yml Template

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

    prod:
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

  target: ephemeral_dev
```

## Key Rules

- Profile name `dbt_fab_spark` must match `profile:` in `dbt_project.yml`
- `ephemeral_dev` → `vdstudio_oauth`, `ephemeral_dep`/`prod` → `fabric_notebook`
- If a target is missing, add it using the template block above with the correct `authentication` value
- `session_id_file` is relative (`./.livy-session-id.txt`), not `/tmp/`
- No `database` field — only `lakehouse` and `lakehouseid`
