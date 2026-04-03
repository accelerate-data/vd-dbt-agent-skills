---
name: fabric-dbt-deployment
description: Creates and deploys dbt job notebooks to Microsoft Fabric. Use when the user wants to deploy dbt models, create a dbt notebook, set up a dbt job, or schedule dbt execution on Fabric.
allowed-tools: "Bash(uv run *), Bash(cat *), Bash(grep *), Bash(python3 *), Bash(mkdir *), Bash(ls *), Read, Write, Edit, Glob"
user-invocable: false
metadata:
  author: accelerate-data
---

# dbt Deployment — Create & Deploy Notebooks

You create **thin Fabric notebooks** that run dbt commands against a Fabric Lakehouse. Each notebook is a single dbt job (e.g. one fact group). All heavy lifting lives in the `vd-dbt-fabricspark` adapter package — the notebook just configures and calls `run_dbt_job()`.

## Core Principles

- **One notebook = one dbt job.** Each logical unit (fact group, test suite, seed) gets its own notebook.
- **Notebooks are thin wrappers.** Three cells: parameters, pip install, execute. No business logic in the notebook.
- **Git repo is source of truth.** Notebooks are committed to git alongside the dbt project. Fabric syncs from git.
- **Adapter package owns the runtime.** The `vd-dbt-fabricspark` package provides `run_dbt_job()` which handles env setup, repo clone, dbt execution, logging, and artifact persistence.

## When to Use This Skill

- User says "deploy my dbt models" or "create a notebook for X"
- User wants to set up a dbt job for scheduling
- After design phase validation, user is ready to move to deployment
- User asks to create notebooks for fact groups

## Notebook Structure

Every generated notebook has exactly 3 code cells:

### Cell 1: Parameters

Flat parameters injected by Fabric Data Pipeline at runtime. These are the knobs the pipeline passes in.

### Cell 2: Install

Single pip install of the adapter package.

### Cell 3: Execute

Import `run_dbt_job` from the adapter, construct config, execute.

## Step-by-Step Workflow

### 1. Gather Information

Before generating a notebook, collect:

| Information | `.env` variable | Hardcode as default? |
|-------------|-----------------|---------------------|
| dbt command | _(from user request)_ | Yes |
| Notebook name | _(derived from command)_ | — |
| Repo URL | `REPO_URL` | Yes |
| Repo branch | _(user input, default "main")_ | Yes |
| GitHub App ID (KV secret name) | `GITHUB_APP_ID` | Yes |
| GitHub Installation ID (KV secret name) | `GITHUB_INSTALLATION_ID` | Yes |
| GitHub PEM secret (KV secret name) | `GITHUB_PEM_SECRET` | Yes |
| Key Vault URL | `AZURE_KEYVAULT_URL` | Yes |
| Lakehouse name | `LAKEHOUSE` | No (leave empty) |
| Lakehouse ID | `LAKEHOUSE_ID` | No (leave empty) |
| Workspace ID | `WORKSPACE_ID` | No (leave empty) |
| Workspace name | `WORKSPACE_NAME` | No (leave empty) |

**Read defaults from the intent's `.env` file.** Run `cat .env` or `grep` for the variables above. All GitHub App fields (`GITHUB_APP_ID`, `GITHUB_INSTALLATION_ID`, `GITHUB_PEM_SECRET`) are **Key Vault secret names** — the adapter fetches the actual values from Azure Key Vault at notebook runtime.

**Default values matter.** These notebooks are meant to be scheduled — the parameter cell defaults are what Fabric uses when no pipeline overrides them. Always bake the command, repo URL, branch, Key Vault URL, and GitHub App secret names as defaults. Lakehouse/workspace coordinates are left empty because they are injected at runtime by the pipeline or set via the notebook's lakehouse attachment.

If the user says "deploy my fact group X", the command is `dbt run --select tag:X`.
If the user says "create a test notebook", the command is `dbt test` (or with selectors).

### 2. Generate the Notebook

Use the Python script below to generate a valid Fabric notebook. **Never hand-write notebook JSON** — the cell `source` field must be an array of strings where each line ends with `\n`.

```bash
python3 -c "
import json, sys, uuid

notebook_name = sys.argv[1]
command = sys.argv[2]
repo_url = sys.argv[3]
repo_branch = sys.argv[4] if len(sys.argv) > 4 else 'main'
github_app_id = sys.argv[5] if len(sys.argv) > 5 else ''
github_installation_id = sys.argv[6] if len(sys.argv) > 6 else ''
github_pem_secret = sys.argv[7] if len(sys.argv) > 7 else ''
vault_url = sys.argv[8] if len(sys.argv) > 8 else ''
description = sys.argv[9] if len(sys.argv) > 9 else f'dbt job: {command}'

notebook = {
    'metadata': {
        'language_info': {'name': 'python'},
        'kernel_info': {'name': 'jupyter'},
        'kernelspec': {
            'name': 'jupyter',
            'display_name': 'Jupyter',
            'language': 'python',
        },
        'microsoft': {'language_group': 'jupyter_python'},
        'dependencies': {
            'lakehouse': {
                'default_lakehouse': '',
                'default_lakehouse_name': '',
                'default_lakehouse_workspace_id': '',
                'known_lakehouses': [],
            }
        },
    },
    'nbformat': 4,
    'nbformat_minor': 5,
    'cells': [
        {
            'cell_type': 'markdown',
            'metadata': {},
            'source': [
                f'# {notebook_name}\n',
                '\n',
                f'{description}\n',
            ],
        },
        {
            'cell_type': 'code',
            'metadata': {'tags': ['parameters']},
            'source': [
                '# Parameters (injected at runtime by Fabric Data Pipeline)\n',
                f'command = \"{command}\"\n',
                f'repo_url = \"{repo_url}\"\n',
                f'repo_branch = \"{repo_branch}\"\n',
                f'github_app_id = \"{github_app_id}\"\n',
                f'github_installation_id = \"{github_installation_id}\"\n',
                f'github_pem_secret = \"{github_pem_secret}\"\n',
                f'vault_url = \"{vault_url}\"\n',
                'lakehouse_name = \"\"\n',
                'lakehouse_id = \"\"\n',
                'workspace_id = \"\"\n',
                'workspace_name = \"\"\n',
                'schema_name = \"dbo\"\n',
            ],
            'outputs': [],
            'execution_count': None,
        },
        {
            'cell_type': 'code',
            'metadata': {},
            'source': [
                '# Install dbt adapter\n',
                '!pip install vd-dbt-fabricspark -q\n',
            ],
            'outputs': [],
            'execution_count': None,
        },
        {
            'cell_type': 'code',
            'metadata': {},
            'source': [
                'from dbt.adapters.fabricspark.notebook import (\n',
                '    run_dbt_job,\n',
                '    DbtJobConfig,\n',
                '    RepoConfig,\n',
                '    ConnectionConfig,\n',
                ')\n',
                '\n',
                'config = DbtJobConfig(\n',
                '    command=command,\n',
                '    repo=RepoConfig(\n',
                '        url=repo_url,\n',
                '        branch=repo_branch,\n',
                '        github_app_id=github_app_id,\n',
                '        github_installation_id=github_installation_id,\n',
                '        github_pem_secret=github_pem_secret,\n',
                '        vault_url=vault_url,\n',
                '    ),\n',
                '    connection=ConnectionConfig(\n',
                '        lakehouse_name=lakehouse_name,\n',
                '        lakehouse_id=lakehouse_id,\n',
                '        workspace_id=workspace_id,\n',
                '        workspace_name=workspace_name,\n',
                '        schema_name=schema_name,\n',
                '    ),\n',
                ')\n',
                '\n',
                'result = run_dbt_job(config)\n',
            ],
            'outputs': [],
            'execution_count': None,
        },
    ],
}

print(json.dumps(notebook, indent=1))
" "$@"
```

**Arguments:** `notebook_name command repo_url [repo_branch] [github_app_id] [github_installation_id] [github_pem_secret] [vault_url] [description]`

**Example:**
```bash
python3 -c "..." \
  "orders_fact_group" \
  "dbt run --select tag:orders" \
  "https://github.com/myorg/myrepo" \
  "main" \
  "12345" \
  "67890" \
  "github-app-pem" \
  "https://my-vault.vault.azure.net/" \
  "dbt job: run orders fact group"
```

### 3. Write Notebook to Repo

Write the notebook into the **parent directory** of the dbt project (the git repo root level), following Fabric's `.Notebook` directory convention:

```
{repo_root}/
├── notebooks/
│   └── {notebook_name}.Notebook/
│       ├── .platform
│       └── notebook-content.ipynb
├── dbt_project.yml
├── models/
└── ...
```

**Directory:** `notebooks/{notebook_name}.Notebook/`

**Files to create:**

1. `notebook-content.ipynb` — the notebook JSON (generated above)
2. `.platform` — Fabric metadata file:

```json
{
    "$schema": "https://developer.microsoft.com/json-schemas/fabric/gitIntegrationPlatformProperties/2.0.0/schema.json",
    "metadata": {
        "type": "Notebook",
        "displayName": "{notebook_name}",
        "description": "{description}"
    },
    "config": {
        "version": "2.0",
        "logicalId": "{generated-uuid}"
    }
}
```

Generate a fresh UUID for each new notebook's `logicalId`.

### 4. Deploy to Fabric

After writing the notebook files to the repo, deploy using the **fabric-cli** skill (available to the main agent):

```bash
# Import the notebook into the Fabric workspace
uv run --env-file .env fab import {workspace_name}.Workspace/{notebook_name}.Notebook \
  -i notebooks/{notebook_name}.Notebook -f
```

If the notebook already exists in Fabric, delete and reimport:

```bash
uv run --env-file .env fab rm {workspace_name}.Workspace/{notebook_name}.Notebook -f
uv run --env-file .env fab import {workspace_name}.Workspace/{notebook_name}.Notebook \
  -i notebooks/{notebook_name}.Notebook -f
```

After import, attach the default lakehouse:

```bash
uv run --env-file .env fab set {workspace_name}.Workspace/{notebook_name}.Notebook \
  -q lakehouse \
  -i '{"known_lakehouses": [{"id": "{lakehouse_id}"}], "default_lakehouse": "{lakehouse_id}", "default_lakehouse_name": "{lakehouse_name}", "default_lakehouse_workspace_id": "{workspace_id}"}'
```

### 5. Test the Deployed Notebook

Run the notebook to validate the deployment:

```bash
uv run --env-file .env fab job run {workspace_name}.Workspace/{notebook_name}.Notebook \
  -P "lakehouse_name:string={lakehouse_name},lakehouse_id:string={lakehouse_id},workspace_id:string={workspace_id},workspace_name:string={workspace_name},schema_name:string=dbo" \
  --timeout 600
```

Since command, repo, and GitHub App credentials are baked into the parameter cell defaults, you only need to pass the lakehouse/workspace coordinates as runtime overrides.

Check the result. If it fails, inspect logs and fix.

## Naming Conventions

| Job Type | Notebook Name | Command |
|----------|--------------|---------|
| Fact group run | `{fact_group}_fact_group` | `dbt run --select tag:{fact_group}` |
| Fact group test | `{fact_group}_fact_group_test` | `dbt test --select tag:{fact_group}` |
| Full project run | `dbt_run_all` | `dbt run` |
| Full project test | `dbt_test_all` | `dbt test` |
| Seed | `dbt_seed` | `dbt seed` |
| Snapshot | `dbt_snapshot` | `dbt snapshot` |
| Custom selector | `dbt_{selector_slug}` | `dbt run --select {selector}` |

The notebook directory name is always `{notebook_name}.Notebook` (Fabric convention).

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Parameter cell missing `"tags": ["parameters"]` | The metadata for the parameter cell **must** include `{"tags": ["parameters"]}` — this is how Fabric identifies injectable parameters |
| Cell `source` is a string, not array | Always use the Python generator script — never hand-write JSON |
| Source lines missing trailing `\n` | The generator handles this — each line must end with `\n` |
| Missing `.platform` file | Always create both `.platform` and `notebook-content.ipynb` |
| Empty defaults for command/repo/GitHub App | These notebooks are scheduled — bake real values as defaults so they work without pipeline parameter overrides |
| Writing notebook inside the dbt project directory | Write to repo root: `notebooks/{name}.Notebook/`, not inside `models/` |
| Forgetting to attach lakehouse after import | Always run `fab set ... -q lakehouse` after import |
| Missing `workspace_name` parameter | Always include `workspace_name` alongside `workspace_id` — the adapter sets both in the environment |

## What This Skill Does NOT Do

- **Scheduling** — that's a post-deployment step, handled separately
- **Refactoring** — split/merge/rename of fact groups is out of scope
- **Design phase validation** — the dbt models should already be validated before deployment
- **vibedata_runtime** — no shared runtime notebook; all logic is in the adapter package
