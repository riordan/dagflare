# Development environment

Prepared October 13, 2026 for research, documentation, and local Cloudflare execution experiments. Dagflare itself remains a design-stage project.

## Current workflow and tools

The repository is `/workspace/dagflare`. Cloud tasks already provide an isolated checkout; use it directly and do not create a Git worktree unless explicitly requested.

Tooling and smoke fixtures are outside the repository so onboarding does not add an application skeleton or modify project dependency declarations:

| Tool or directory                  | Prepared version or purpose                                        |
| ---------------------------------- | ------------------------------------------------------------------ |
| Node / npm                         | 24.19.0 / 11.9.0 in the cloud runtime                              |
| Python                             | 3.12.14 in the cloud runtime                                       |
| Wrangler                           | 4.150.0; includes Basin command groups                             |
| Prettier / markdownlint-cli2       | 3.9.10 / 0.23.3                                                    |
| dbt-core / dbt-duckdb              | 1.12.5 / 1.11.0                                                    |
| DuckDB / PyIceberg                 | 1.5.6 / 0.12.0                                                     |
| PyArrow                            | 26.0.0                                                             |
| `/workspace/.dagflare-tools`       | Node tool dependencies and npm lockfile                            |
| `/workspace/.dagflare-python`      | Isolated Python environment                                        |
| `/workspace/.dagflare-environment` | Activation, pinned Python requirements, checks, and smoke fixtures |
| `/workspace/.dagflare-state`       | Local database/object/workflow state and generated artifacts       |
| `/workspace/.cache`                | Writable npm/uv caches                                             |

The saved environment installation script recreates these helpers and dependencies. It does not deploy resources, write project source, or publish the environment. Existing lockfiles are reused; tool refreshes require stopping dev processes first.

Activate tools in each new shell:

```bash
source /workspace/.dagflare-environment/activate.sh
cd /workspace/dagflare
```

This sets tool paths, writable caches, and local configuration paths without changing HOME or printing credentials. Wrangler and dbt anonymous usage collection are disabled for this environment.

## Documentation checks

```bash
prettier --check '**/*.md'
markdownlint-cli2 --config /workspace/.dagflare-environment/markdownlint.jsonc '**/*.md'
python /workspace/.dagflare-environment/check-docs.py
```

The Markdown configuration permits long research tables and delegates table alignment to Prettier. The link check validates local destinations; it does not claim that external websites are reachable or that a vendor's live page matches its pinned source. It rejects local anchor links until anchor validation is added.

## Local Cloudflare runtime smoke test

The fixture tests development prerequisites. It is outside the checkout and is not the Dagflare product API.

Run the complete functional check, which starts the local server, checks outputs, restarts it, verifies persistence, and stops the processes it started:

```bash
source /workspace/.dagflare-environment/activate.sh
python /workspace/.dagflare-environment/smoke/run.py
```

For interactive experimentation, run this in a terminal and stop it when done:

```bash
source /workspace/.dagflare-environment/activate.sh
exec wrangler dev --local --ip 127.0.0.1 --port 8787 \
  --config /workspace/.dagflare-environment/smoke/wrangler.json \
  --persist-to /workspace/.dagflare-state/smoke \
  --show-interactive-dev-session=false
```

The check verifies HTTP readiness, launches a Workflow, waits for completion, and confirms the expected D1 row and R2 object. It is more than a port check. Local state persists under the specified directory. Stop only processes you started; dependency refreshes refuse to proceed while a prepared Wrangler process is active.

Wrangler reports a fallback for Request.cf metadata because the metadata fetch is blocked in this environment. Local geolocation/request metadata is therefore not representative of production. The functional storage/Workflow check succeeds.

## Local dbt smoke test

```bash
source /workspace/.dagflare-environment/activate.sh
dbt build \
  --project-dir /workspace/.dagflare-environment/dbt-smoke \
  --profiles-dir /workspace/.dagflare-environment/dbt-smoke \
  --target-path /workspace/.dagflare-state/dbt-target \
  --log-path /workspace/.dagflare-state/dbt-logs
python /workspace/.dagflare-environment/check-dbt.py
```

The fixture loads three orders, aggregates two customer totals, and runs uniqueness, non-null, and expected-total checks. Result verification inspects the current dbt artifacts, manifest dependency, and actual rows. The run passed five operations: one seed, one model, and three tests, with no errors, warnings, or skips. It also passed on repeat execution.

This uses local DuckDB. It does not validate dbt materializations against Basin Catalog. Live integration is [Gate C](design/validation-plan.md).

## GitHub and external access

The existing HTTPS Git route can read the repository. Use that route for Git operations; unset raw token variables or a separate CLI login prompt are not proof that Git access is missing. GitHub API access is a separate operation and was blocked by the network policy during preparation.

The draft includes additions for `developers.cloudflare.com`, `blog.cloudflare.com`, `docs.dagster.io`, `docs.prefect.io`, and `api.github.com`. Saving these additions does not apply them to a running instance. Research used official Git source repositories over the already available route; the [source register](research/sources.md) preserves exact revisions.

No Cloudflare credentials are required for documentation or the local checks. Cloudflare API token/account and Basin query-token variables were absent during inspection. Live account work needs securely configured, scoped credentials and endpoint access for the specific operation; no secret values should be sent in chat or stored in Git. The current environment draft does not require Cloudflare secrets merely to work on the design docs.

## Validation boundaries

Verified: documentation formatting/lint/local links; native Git transport as reported by the publishing result; local Worker, Workflow, D1 and R2 behavior; local dbt artifacts and output; repeatable dependency installation and smoke commands. Docker CLI and a responsive engine are present.

Unverified: deployed Dagflare behavior, real Basin ingestion/query/writes, Cloudflare Container builds/deployment, Access/RBAC, live GitHub API integration, account portability, and restoration in a new cloud task. These remain explicit experiments, not current setup blockers for the docs workflow.

Environment publication is a user action in environment settings. A saved draft and the current local checks do not establish publication or a successful fresh-task restore. Processes must be started again in a new task; the saved startup instructions describe the necessary commands.
