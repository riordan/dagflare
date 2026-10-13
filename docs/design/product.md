# Product scope and user workflows

Status: proposed. Owner requirement: every team deploys Dagflare into its own Cloudflare account; no shared infrastructure or central Dagflare service. The first deployment may serve one team in the project owner's account.

## Product purpose

Dagflare gives a data team one place to answer: what data exists, what produces it, whether it is usable, what changed, and how to update it safely. Basin supplies the lakehouse primitives. Dagflare supplies the asset model and connects day-to-day operation to code review.

The initial user is a small technical team building event analytics on Cloudflare. Its platform engineer deploys the application and integrations; its data engineer writes SQL/dbt definitions; its analyst explores permitted assets; its operator investigates freshness and failures. This is an initial product hypothesis, not validated customer research.

## The first complete workflow

1. A platform engineer deploys Dagflare and configures the team's identity, Git repository, and permitted Cloudflare resources.
2. Dagflare discovers an existing Basin Catalog and imports its namespaces/tables as observed assets.
3. A data engineer declares a source asset and a derived model in Git, with owner, dependency, executor, checks, and freshness expectation.
4. A PR shows the definition diff, changed graph, affected downstream assets, resource plan, and validation results.
5. A reviewed definition is promoted. The engineer runs it manually, then enables its schedule through a separately audited operational action.
6. The UI shows committed data, relevant Iceberg snapshot, model checks, freshness, run history, and definition SHA.
7. When a run fails, an operator can distinguish runner failure, transformation failure, delayed ingestion, dropped records, and failed checks, then retry only the appropriate work.

The first workflow is complete only when another team can deploy the same artifact into another account without depending on the project owner's account or a Dagflare-hosted service.

## Navigation and interaction model

| Surface      | Main question                     | Required first-release behavior                                                                       |
| ------------ | --------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Overview     | What needs attention?             | Freshness/check failures, blocked runs, delayed ingestion, observation age                            |
| Catalog      | What data do we have?             | Search/filter by kind, namespace, owner, environment, health; show discovered versus managed          |
| Asset detail | Can I trust and use this asset?   | Schema, description, owner, dependencies, definition SHA, recent observations/materializations/checks |
| Lineage      | What does this change affect?     | Upstream/downstream graph, selection, dependency type, declared/observed provenance                   |
| Runs         | What executed and what happened?  | State, steps/models, attempts, logs/artifacts, output snapshots, retry/cancel controls                |
| Automation   | Why did or didn't it run?         | Schedule/policy, next expected tick, skip/block reason, active state, evaluation history              |
| Changes      | What is under review or deployed? | PR/MR links, base/head SHA, graph diff, validation, promotion history                                 |
| Settings     | Who can act and where?            | Roles, integrations, resource scope, retention, account quotas, installed version                     |

Every surface shows the current environment and definition version. Production and preview should be visually explicit. Operational controls show their effect before execution; configuration editing points to the resulting Git review.

## Asset and health semantics

An asset has a stable key, an execution/observation kind, dependencies, an owner, and an environment-specific physical binding. Definitions are versioned. Events and observations are historical evidence. Resources can be discovered without transferring their management to Dagflare.

Separate these concepts in the UI:

- **Ingestion acceptance:** a producer request was acknowledged.
- **Processing/delivery:** pipeline metrics show decoded or dropped records and delivered files.
- **Observation:** Dagflare saw a table/schema/snapshot at a known time.
- **Materialization:** a finite run produced a verified, committed output.
- **Check:** an assertion evaluated against a particular output/version.
- **Freshness:** measured data coverage meets a declared expectation, with evidence age shown.

An observed snapshot with unknown lineage should not fabricate a successful Dagflare run. Missing measurements produce “unknown,” not “healthy.” Maintenance should not masquerade as new business data.

## Declarative definitions

The initial definition format should be a versioned JSON or YAML manifest with JSON Schema validation. SQL and dbt code remain in their normal files. The compiled manifest supplies a deterministic asset DAG, execution adapters, permissions/resource plan, and policy references. The exact public schema is a follow-up design task.

Required fields are stable asset identity, kind, dependency keys, code/source reference, physical target mapping, and adapter configuration. Owners, descriptions, tags, partition definition, checks, freshness, and automation policies enrich it. Credentials appear only as references to team-owned secret bindings.

The UI can propose edits, but the authoritative save becomes a Git commit/PR. Emergency operational controls such as pausing schedules have an audit record and can override a definition without silently rewriting it. Importing a discovered table creates an observation; adopting it as managed requires an explicit plan.

## MVP scope

| Capability   | First complete version                                                            | Later expansion                                                   |
| ------------ | --------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Deployment   | One account, one team, independently repeatable installer                         | Multiple projects/environments in the same account                |
| Catalog      | Basin namespaces/tables, schema, owners, bounded previews                         | Rich search, documentation generation, broader connectors         |
| Lineage      | Table/asset dependencies from definitions and dbt manifest                        | Column lineage with dialect-aware parsing and confidence          |
| Execution    | Manual finite runs, durable steps, one proven transformation adapter              | More adapters, broader concurrency policies                       |
| dbt          | Import model graph and test outcomes; execute only validated materializations     | Incremental strategies, snapshots, advanced selection/state/defer |
| Scheduling   | UTC schedules plus clear expected-tick semantics                                  | Timezones/DST, dependency-aware policies, event absence alerts    |
| Partitions   | Reserve partition keys in the model; support one daily-partition path when proven | Arbitrary mappings, dynamic/multi-dimensional partitions          |
| Checks       | Results attached to the committed output version                                  | Contracts, blocking policy algebra, richer quality integrations   |
| Git workflow | GitHub first: plan/diff/validation/promotion tied to SHA                          | GitLab MR integration, writable branch sandboxes                  |
| Identity     | Cloudflare Access authentication; Viewer/Operator/Editor/Admin authorization      | Fine-grained asset/project scopes and directory provisioning      |
| Operations   | Structured history, artifacts, audit, basic retention/backup                      | Cost attribution, notifications, dashboards, recovery tooling     |

Start with UTC scheduling explicitly; do not silently interpret a team's local-time cron as UTC. Add timezone scheduling only after its DST and missed-tick semantics are tested.

## Deferred scope

A broad ETL connector marketplace, full CDC, notebook environment, arbitrary Python SDK compatibility, a visual drag-and-drop pipeline builder, multi-cloud compute management, automatic production data clones, warehouse-grade writes through Basin SQL, enterprise compliance certifications, and a hosted Dagflare SaaS are outside the first release.

These are scope choices, not statements that Basin or a future Dagflare can never support them.

## Acceptance criteria

- An imported catalog table is visible with its actual schema and last observation time; missing permissions yield a useful diagnosis.
- A declared graph rejects cycles and missing dependencies and produces the same definition digest on repeat compilation.
- A run reports per-model/step results, committed output identity, checks, and artifacts; duplicate triggers cannot create duplicate logical materializations.
- A failed check remains visible alongside a successful commit, and dependent execution follows an explicit blocking policy.
- A PR's asset impact is anchored to recorded base/head SHAs; previews cannot write to production.
- An operator can pause schedules without editing Git, and that action remains audited across a definition rollback.
- Viewer credentials cannot run jobs, read secrets, bypass sample restrictions, or call privileged resource APIs.
- A fresh installation in a second account operates independently of the first.

## How to assess the prototype

Measure time to discover a table, diagnose a deliberately broken run, identify downstream impact, and promote a reviewed change. Also measure observation lag, ingestion-to-query latency, runner startup/runtime, cost per representative workload, and recovery after a process or deployment interruption. Numeric SLOs should follow experiments rather than precede them.

The next step is the technical proof described in the [validation plan](validation-plan.md), followed by a clickable UI prototype grounded in the actual event/data model.
