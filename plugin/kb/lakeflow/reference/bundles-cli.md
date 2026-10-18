# Declarative Automation Bundles and the Databricks CLI

> **Purpose**: Bundle configuration, deployment modes, variables, CLI versions and the direct deployment engine (Terraform engine removed in CLI 1.20.0), commands, and CI authentication
> **Confidence**: High
> **MCP Validated**: 2026-10-17

Declarative Automation Bundles (formerly known as Databricks Asset Bundles; acronym DABs kept) describe
Databricks resources (jobs, pipelines, dashboards, apps, schemas, volumes, ...) as YAML or Python in
the same repository as the code, and the Databricks CLI deploys them per target.

---

## 1. CLI versions (as of 2026-10-17)

| Version | Date | Bundle-relevant change |
|---------|------|------------------------|
| 0.218.0 | - | Minimum CLI for bundles |
| 0.275.0 | - | Minimum CLI for Python support (`databricks-bundles` package) |
| 0.279.0 | - | Two deployment engines available: `terraform` and `direct` |
| 1.3.0 | - | New bundles default to the direct engine |
| 1.8.0 | - | `bundle.engine: direct` opt-in for existing bundles; clients older than 1.8.0 reject state written by 1.17.0+ |
| 1.14 | - | Bundles with Terraform state auto-migrate to direct on deploy |
| 1.19.x | 2026-09-30 | Last line that can still deploy with Terraform (`engine: terraform`) |
| **1.20.0** | **2026-10-07** | **Terraform deployment engine removed** |
| **1.21.0** | **2026-10-14** | Latest: multipart `workspace/import` uploads, retries on 5xx/408 for deploy/sync uploads, `bundle deployment bind` state fix |

### What the 1.20.0 removal means in practice

- `bundle.engine: terraform` and `DATABRICKS_BUNDLE_ENGINE=terraform` now fail. To keep Terraform,
  pin CLI 1.19.x (a stopgap; new resources and fixes land only on the direct engine).
- A bundle with Terraform state is migrated to the direct engine before deploying. If that migration
  fails, the deploy is aborted (it no longer falls back to Terraform). Since 1.15 the migration first
  runs a plan against the converted state and abandons the migration if that plan fails.
- No Terraform binary, provider download or registry access is needed in CI (firewall/proxy issues
  around the Terraform provider disappear).
- Behavior differences to review before upgrading a CI runner:
  - The direct engine stores only the local configuration snapshot as state and diffs local config
    against the last snapshot, then remote against the snapshot.
  - **Removing a field from YAML resets it to the resource default** on the next deploy (Terraform
    left the old value in place). Keep every setting you rely on explicit.
  - `${resources.*}` substitutions resolve from local config first, then from the remote resource.
- New capabilities: `databricks bundle plan -o json` with per-field reasons, `bundle deploy --plan
  plan.json` to apply an approved plan, resources only supported by the direct engine (cluster
  policies, UC catalogs, external locations, UC secrets, Genie spaces, instance pools, job runs,
  AI Search endpoints, and more), `lifecycle.started` for apps/clusters/SQL warehouses.
- Manual migration ahead of an upgrade: `databricks bundle deployment migrate` (Public Preview),
  then `databricks bundle plan` should show no unexpected actions, then deploy.

### Install and pin

```bash
# Latest
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
# Pinned (recommended in CI)
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/v1.21.0/install.sh | sh
databricks --version
```

GitHub Actions: `uses: databricks/setup-cli@v1.21.0` (or `@main` with `with: {version: 1.21.0}`).
macOS: `brew tap databricks/tap && brew install databricks`. Windows: WinGet or Chocolatey.

Guard the bundle against old CLIs:

```yaml
bundle:
  name: my_bundle
  databricks_cli_version: ">= 1.20.0"   # validate fails outside the range
```

## 2. Configuration mappings

| Mapping | Purpose |
|---------|---------|
| `bundle` | `name` (required), `databricks_cli_version`, `engine` (only `direct` on 1.20.0+), `git`, `deployment.lock` |
| `include` | Extra YAML files (globs), e.g. `resources/*.yml` |
| `variables` | Custom variables (`default`, `description`, `type: complex`, `lookup`) |
| `workspace` | `host`, `profile`, `root_path`, `file_path`, `artifact_path` |
| `resources` | `jobs`, `pipelines`, `dashboards`, `apps`, `schemas`, `volumes`, `registered_models`, ... |
| `targets` | Per-environment overrides: `mode`, `default`, `workspace`, `variables`, `resources`, `presets`, `run_as`, `git` |
| `presets` | `name_prefix`, `pipelines_development`, `trigger_pause_status`, `jobs_max_concurrent_runs`, `tags` |
| `run_as` | `user_name` or `service_principal_name` (resources also accept `group_name`) |
| `permissions` | Top-level levels: `CAN_VIEW`, `CAN_RUN`, `CAN_MANAGE` |
| `artifacts` | Build Python wheels/JARs before deploy |
| `sync` | `include` / `exclude` / `paths` for uploaded files |
| `scripts` | Named commands run with `databricks bundle run <script>` |
| `python` | Python resource loaders (`resources:load_resources`) |
| `experimental` | `immutable_folder` (read-only deploy folder, CLI 1.6.0+), `deployment_history` (CLI 1.16.0+) |

Bundle identity is `root_path` (default `~/.bundle/${bundle.name}/${bundle.target}`): same name,
target and deployer means the same deployment.

## 3. Variables and substitutions

| Syntax | Resolved |
|--------|----------|
| `${var.<name>}` | Deploy time |
| `${bundle.name}`, `${bundle.target}` | Deploy time (`${bundle.environment}` is legacy) |
| `${workspace.current_user.userName}` / `.short_name` / `.domain_friendly_name` | Deploy time |
| `${workspace.host}`, `${workspace.file_path}`, `${workspace.root_path}` | Deploy time |
| `${resources.jobs.<key>.id}`, `${resources.pipelines.<key>.id}` | After the resource exists |
| `{{job.parameters.x}}` etc. | Job run time (dynamic value references) |

Variable precedence (first match wins): `--var="k=v"` > `BUNDLE_VAR_<name>` env var >
`.databricks/bundle/<target>/variable-overrides.json` > `targets.<t>.variables` > `default`.
Use the same mechanism at deploy and run. Variables are deployment-time; use job parameters for
per-run values.

## 4. Deployment modes

| | `development` | `production` |
|---|---|---|
| Names | `[dev <short_name>]` prefix + `dev` tag on jobs/pipelines | unchanged |
| Pipelines | `development: true` | must be `development: false` |
| Schedules/triggers | paused (`pause_status: UNPAUSED` on a resource overrides) | unchanged |
| Jobs | concurrent runs enabled | unchanged |
| Lock | disabled (`bundle.deployment.lock.enabled: true` re-enables) | enabled |
| Compute override | `--cluster-id` / `cluster_id` allowed | not allowed |
| Checks | - | Git branch must equal `git.branch` (if set; `--force` skips); without a service principal: `run_as`/`permissions` required and paths must not be user-specific |

Presets override mode defaults; explicit resource settings override presets.

## 5. Commands

| Command | Key flags |
|---------|-----------|
| `bundle init [TEMPLATE]` | `--output-dir`, `--config-file`, `--branch` |
| `bundle validate` | `-t`, `--var`; warns on unknown properties |
| `bundle plan` | `-t`, `--select`, `-o json` (direct engine) |
| `bundle deploy` | `-t`, `--auto-approve`, `--fail-on-active-runs`, `--force`, `--force-lock`, `--plan <file>`, `--select` (recommended for dev workspaces only), `-c/--cluster-id` |
| `bundle run [KEY]` | jobs: `--params k=v,...`, `--only a,+b,c+`; pipelines: `--refresh`, `--refresh-all`, `--full-refresh`, `--full-refresh-all`, `--validate-only`; `--no-wait`, `--restart`; `-- <cmd>` runs a script with bundle auth |
| `bundle summary` / `bundle open` | Show deployed resources / open in the UI |
| `bundle destroy` | `--auto-approve`; `lifecycle` can protect resources |
| `bundle generate job\|pipeline\|dashboard\|app\|genie-space` | `--existing-job-id` / `--existing-pipeline-id`, `--bind` |
| `bundle deployment bind KEY ID` / `unbind KEY` | Adopt or release existing resources (bind keeps data, no MV recompute) |
| `bundle deployment migrate` | Terraform state to direct engine (Public Preview) |
| `bundle schema`, `bundle sync` | JSON schema; file sync |

## 6. Authentication for automation

| Scenario | Recommended | Environment |
|----------|-------------|-------------|
| Local, attended | OAuth U2M (`databricks auth login`), profiles via `-p` | `~/.databrickscfg` |
| CI, unattended | OAuth M2M with a service principal | `DATABRICKS_HOST`, `DATABRICKS_CLIENT_ID`, `DATABRICKS_CLIENT_SECRET` |
| GitHub Actions without secrets | Workload identity federation | `DATABRICKS_AUTH_TYPE: github-oidc`, `DATABRICKS_HOST`, `DATABRICKS_CLIENT_ID` |
| Fallback | PAT of a service principal | `DATABRICKS_TOKEN` |

Use one service principal with access to all target workspaces so CI uses one set of variables;
deploy production with `run_as` set to a service principal. Do not hard-code hosts or client IDs
when the same bundle serves several workspaces.

## 7. Python for bundles

- Package `databricks-bundles` (CLI 0.275.0+), environment usually managed with `uv`.
- `databricks.yml`: `python: { venv_path: .venv, resources: ["resources:load_resources"], mutators: [...] }`.
- `load_resources(bundle)` returns `Resources`; mutators (for example `@job_mutator`) adjust
  resources defined in YAML or Python.
- Some Beta resources (for example `postgres_*`) are YAML-only.

## Sources

- https://docs.databricks.com/aws/en/dev-tools/bundles/
- https://docs.databricks.com/aws/en/dev-tools/bundles/settings
- https://docs.databricks.com/aws/en/dev-tools/bundles/reference
- https://docs.databricks.com/aws/en/dev-tools/bundles/variables
- https://docs.databricks.com/aws/en/dev-tools/bundles/deployment-modes
- https://docs.databricks.com/aws/en/dev-tools/bundles/direct
- https://docs.databricks.com/aws/en/dev-tools/bundles/permissions
- https://docs.databricks.com/aws/en/dev-tools/bundles/authentication
- https://docs.databricks.com/aws/en/dev-tools/bundles/python/
- https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands
- https://docs.databricks.com/aws/en/dev-tools/cli/install
- https://docs.databricks.com/aws/en/dev-tools/auth/oauth-m2m
- https://docs.databricks.com/aws/en/dev-tools/ci-cd/github
- https://github.com/databricks/cli/releases/tag/v1.21.0
- https://github.com/databricks/cli/releases/tag/v1.20.0
- https://github.com/databricks/cli/releases/tag/v1.19.0
- https://github.com/databricks/cli/releases/tag/v1.17.0
- https://github.com/databricks/cli/releases/tag/v1.15.0
- https://github.com/databricks/setup-cli
