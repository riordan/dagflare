# Dagster and Prefect product assessment

Evidence date: October 13, 2026. The [source register](sources.md) links vendor-maintained documentation at exact revisions. Dagflare is inspired by these products; compatibility with their APIs is not proposed for the initial release.

## Dagster's organizing idea: assets

Dagster treats an asset as a persistent object such as a table, file, or model. An asset definition says what should exist and how to compute it; materialization executes that definition and saves a result. This distinction gives users a view of the data they care about, rather than only a list of tasks that happened to run. Asset dependencies form a graph spanning systems and integrations. External assets and observations can describe data produced elsewhere. [DG01, DG02](sources.md)

For Dagflare, a table in Basin Catalog is an obvious asset, but a stream, continuous ingestion pipeline, exported dataset, dbt model, and serving endpoint may also need identities. Avoid making every resource a runnable asset. A stream is a resource whose health is observed; a finite batch model is something users can materialize. The graph should state the relationship and execution kind.

### Catalog and lineage

Dagster's asset UI connects definitions, tags, owners, schema, materialization history, and upstream/downstream relationships. The current enhanced catalog guide is explicitly a Dagster+ feature and includes search by asset key, compute kind, group, code location, tags, and owners, plus dashboards and global lineage. Basic OSS asset experiences should not be confused with every enhanced catalog feature. [DG03](sources.md)

Column lineage is emitted as metadata and can be populated from dbt. The inspected docs explicitly restrict column-lineage visualization to Dagster+. Declared dependency lineage, parsed SQL/dbt lineage, and runtime evidence are distinct sources; Dagflare should preserve their provenance and confidence instead of inventing exact dependencies from incomplete SQL. [DG04](sources.md)

The useful product pattern is to let a user move from “this table is stale” to its upstream asset, responsible code, failed run, and owner without changing mental models. Dagflare should make the catalog, graph, and run view share stable asset identities, filters, and environment context.

### Partitions, backfills, checks, and health

Dagster partitions model independently executable slices of an asset. Backfills can target a subset of historical partitions after a logic change or first-time deployment. Dependencies can map partitions between assets, and backfill policies determine batching. This is substantially more than a date picker attached to a rerun button. [DG05, DG06](sources.md)

Asset checks attach data-quality results to assets. Their execution semantics matter: a failed check can describe invalid data even when the transformation successfully committed it. Dagflare should expose computation outcome and check outcome independently, with blocking checks used explicitly in downstream policies. It must not silently rewrite a committed table because a check later fails. [DG07](sources.md)

Dagster+ asset health combines the latest materialization, freshness, and check status. Current freshness docs discuss preview opt-in and distinguish newer policy behavior from deprecated APIs. Dagflare should adopt the user concept but define its own measurable statuses: unknown, healthy, stale, degraded, and failed, with a displayed reason and observation timestamp. A new table snapshot is not sufficient evidence that event-time freshness meets an SLA. [DG08, DG09](sources.md)

### Scheduling and declarative automation

Dagster provides schedules for regular jobs, sensors for events or polling, and declarative automation based on asset state and dependencies. Schedules support timezones and document DST behavior. Sensors have cursors, deduplication run keys, evaluation histories, and skip reasons. Declarative conditions include updating after upstream changes, waiting for dependencies, and updating on cron after upstream data becomes available. [DG10–DG12](sources.md)

These are useful interaction patterns for Dagflare: users should see why a run was requested, skipped, delayed, or blocked. The first implementation should use a small, documented policy vocabulary rather than claim compatibility with Dagster's entire automation-condition algebra. Suggested initial policies are manual, scheduled, and after-upstream-commit, with explicit freshness and check requirements.

### dbt integration

Dagster's dbt integration models individual dbt models, seeds, and snapshots as assets. It supports selecting subsets, correlating per-model failures/logs/history, and linking dbt models to non-dbt upstream/downstream assets. The current docs describe a DbtProjectComponent and cached project state; dbt Fusion support is called preview and lacks some metadata/lineage features. [DG13](sources.md)

The important contract is artifact ingestion, not merely spawning `dbt build`. Dagflare must parse `manifest.json`, correlate `run_results.json` and structured events, preserve model selection and adapter/version details, and report failed or skipped models accurately. Compile-time dependencies and committed runtime outputs must be reconciled. No model should show as materialized solely because the overall process exited zero.

### Versioning and Git review

Dagster distinguishes code versions and data versions. Code changes, dependency changes, and changes to upstream data can make an asset unsynced. Its docs warn that this status is not automatically transitive in the way users may assume: a downstream asset becomes unsynced when its consumed upstream version differs from the current materialized version. [DG14](sources.md)

Dagster+ branch deployments create ephemeral deployments for PRs/MRs and show changed assets. Change tracking includes new assets, code versions, dependencies, partition definitions, tags, and metadata. It can compare against a snapshot of the base or the current base state. The data-isolation guide separately configures cloned schemas and credentials; a branch deployment is not automatically a safe data clone. [DG15, DG16](sources.md)

Dagflare should make the review base explicit: repository base SHA, deployed definition snapshot, PR head SHA, and observed data snapshot are different anchors. UI previews can exist without data writes. Any writable preview must have separate resources and credentials. Git changes should remain authoritative even when initiated from the UI.

### Team identity and deployment

Dagster+ documents organization/deployment roles, custom roles, teams, SSO, and audit logging, with plan-specific availability. The documented role hierarchy includes Viewer, Launcher, Editor, Admin, and Organization Admin. Dagflare should start with similarly understandable roles, but independently enforce permissions at its own API and runner boundaries. [DG17](sources.md)

Dagster OSS deployment requires long-running webserver, daemon, and code-location services plus instance storage and run execution infrastructure. These are legitimate mature-engine capabilities, but hosting them is a distinct architecture choice. Cloudflare Workers are not a drop-in host for the Python daemon or user-code server. [DG18](sources.md)

## Prefect's organizing idea: executable workflows

Prefect flows are decorated Python functions. Tasks provide observed states, retries, caching, concurrency, and timeouts while permitting dynamic control flow. Deployments describe where, when, and how a flow executes; work pools bridge orchestration and execution infrastructure. These patterns are valuable for heterogeneous, imperative jobs that do not map cleanly to a static asset graph. [PF01–PF04](sources.md)

Prefect's state histories distinguish scheduled, pending, running, completed, failed, cancelled, and infrastructure failures. This distinction should inform Dagflare's run interface: queued work is different from a stalled runner, an invalid query, or a failed data check. Cancellation should communicate whether the executor stopped and whether data already committed.

Prefect schedules support cron, intervals, and recurrence rules. Automations can react to events or the absence of an expected event and can launch runs, pause schedules/pools, cancel flows, or notify operators. Dagflare can adopt explicit trigger explanations and missing-event detection without reproducing every schedule type initially. [PF05, PF06](sources.md)

### Prefect assets and deployment history

Current Prefect documentation includes asset materialization, metadata, ownership, and lineage. Asset metadata can be updated at execution time, and explicit dependencies supplement what task execution infers. The inspected asset guides carry Cloud badges; verify current OSS/Cloud boundaries before considering them reusable self-hosted functionality. Prefect should not be characterized as entirely task-only. [PF07, PF08](sources.md)

Prefect Cloud deployment versions preserve many configuration properties, including code-source metadata, and support promoting/rolling back a live version. The guide explicitly says schedules, pause/disable state, and concurrency settings are not versioned with those deployment properties. Creating a new version makes it live by default. This illustrates why operational controls and definition versions need separate histories in Dagflare. [PF09](sources.md)

Prefect Cloud documents RBAC, custom roles, service accounts, SSO, and SCIM with paid-tier qualifications. Do not assume these capabilities are included when running Prefect OSS, and do not introduce a Prefect Cloud dependency into a self-hosted Dagflare installation. [PF10](sources.md)

## Comparison and what to adopt

| Area                | Dagster reference                                    | Prefect reference                                                  | Dagflare proposal                                                |
| ------------------- | ---------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------- |
| Primary abstraction | Persistent assets and their dependencies             | Executable flows/tasks; current asset support also exists          | Asset graph with explicit execution adapters                     |
| Lineage             | Declared graph, integration metadata, column lineage | Task/materialization relationships and explicit asset dependencies | Declared and observed lineage with provenance                    |
| Catalog             | Asset discovery, metadata, history, health           | Asset metadata alongside workflow operations                       | Basin discovery enriched with owners, contracts, freshness, runs |
| Automation          | Schedules, sensors, dependency-aware conditions      | Schedules and event/absence-based automations                      | Small deterministic policy vocabulary with explanations          |
| Partitions          | Rich partition mapping and backfill policies         | Workflow parameters and orchestration patterns                     | Explicit daily partitions first; bounded backfills later         |
| dbt                 | Per-model assets and artifacts                       | Flow-based dbt execution/integrations                              | Per-model artifact import plus a validated Iceberg writer        |
| Execution           | Run launchers and compute integrations               | Work pools and workers                                             | Cloudflare Workflows coordinating account-owned runners          |
| Git review          | Dagster+ branch deployment and impact graph          | Cloud deployment versions and source metadata                      | SHA-bound plan, graph diff, isolated optional sandbox, promotion |
| Identity            | Dagster+ RBAC/SSO/audit                              | Prefect Cloud RBAC/SSO/service identities                          | Team Access identity plus app-level RBAC in each account         |
| Data rollback       | Engine/data-specific behavior                        | Deployment rollback does not necessarily restore data              | Separate code promotion, operational state, and data recovery    |

## Alternatives and decision criteria

**Cloudflare-native Dagflare** has the best fit for the required independent-account deployment and can make Basin concepts first-class. It also means implementing catalog projections, admission, policies, backfill semantics, and Git review. Limit the first release rigorously.

**Dagster OSS plus Basin adapters** reuses the richest asset engine. Its feasibility depends on long-running services, persistent storage, reliable daemon leadership, and runner lifecycles on Cloudflare Containers. It should remain an alternative until those are tested; enhanced Dagster+ features are not a free inheritance.

**Prefect OSS plus Basin adapters** reuses a flexible execution system. It may fit imperative pipelines, but Dagflare would still need an asset-centric UI and independently evaluated self-hosted asset, identity, and Git workflows.

**A catalog-only front end** provides quick discovery and previews, but does not meet the proposed orchestration, Gitflow, and declarative-assets scope. It is a useful first UI slice only if the architecture preserves the eventual run/event model.

Recommendation: proceed with the Cloudflare-native design as a proposal, then make the dbt/Iceberg runner and account-portability experiments the first technical gates. Reassess the engine choice if those experiments reveal requirements that a mature orchestration engine can satisfy more reliably.
