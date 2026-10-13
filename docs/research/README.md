# Research summary

Research date: October 13, 2026. Deployment requirement: a complete, self-hosted Dagflare installation in each team's Cloudflare account. The first prototype may run in the project owner's account.

## What the evidence establishes

Cloudflare Basin is the current name for Cloudflare Data Platform. Cloudflare Pipelines, R2 Data Catalog, and R2 SQL have become Basin Pipelines, Basin Catalog, and Basin SQL. The retrieved documentation labels all three products generally available, while preserving some legacy commands, APIs, and metric identifiers. This is documented current naming, not an inferred product roadmap. [CF01–CF04](sources.md)

Basin provides a credible data plane: buffered event ingestion, stateless SQL transformations during ingestion, R2-backed Iceberg tables, automatic maintenance, and distributed analytical queries. It also already provides dashboard workflows for creating pipelines, exploring catalogs, monitoring metrics, and inspecting maintenance runs. Dagflare should add a coherent operational and development lifecycle rather than merely recreate these screens. [CF05–CF18](sources.md)

Dagster provides the strongest reference for an asset-oriented experience: declarative asset definitions, dependency graphs, partition-aware materialization, checks, freshness, dbt model integration, and lineage. Dagster+ adds managed features such as branch deployments, change tracking, RBAC, SSO, and enhanced catalog experiences. Those managed features should not be presented as capabilities automatically inherited from Dagster OSS. [DG01–DG18](sources.md)

Prefect provides a useful execution reference: dynamic Python flows, tasks, explicit state histories, retries, concurrency controls, work pools, and event-driven automation. Current Prefect documentation also describes assets and lineage; treating Prefect as having no asset model would be outdated. Several asset UI, identity, and deployment-history capabilities are documented specifically for Prefect Cloud. [PF01–PF10](sources.md)

## Findings that change the architecture

| Finding                                                                        | Consequence for Dagflare                                                                                  |
| ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| Basin SQL supports analytical queries but no DDL or DML                        | Do not promise native dbt execution against Basin SQL; use a write-capable Iceberg engine                 |
| DuckDB 1.4+ writes to Iceberg REST catalogs in Cloudflare's documented example | Evaluate a Cloudflare Container running dbt-duckdb; prove materialization compatibility before committing |
| Pipeline transforms are stateless                                              | Batch joins, aggregates, incremental models, and historical recomputation need a different executor       |
| Structured stream schemas cannot be changed in place                           | Treat schema changes as resource migrations with explicit cutover and rollback plans                      |
| Invalid events can be accepted and later dropped during processing             | Show ingest acceptance, decode failures, sink delivery, and table visibility as distinct states           |
| Catalog sinks cannot be created for existing Iceberg tables                    | Initial import and resource replacement cannot assume arbitrary sink reattachment                         |
| Iceberg sink rolling has a minimum 60-second interval                          | Freshness guarantees must include ingestion-to-query latency and measurement uncertainty                  |
| Pipeline quotas are currently 20 streams, sinks, and pipelines per account     | Default PR validation to plans and fixtures; provision bounded sandboxes only when requested              |
| Basin Catalog currently excludes non-default R2 jurisdictions                  | Do not claim EU jurisdiction support; compliance requirements can block adoption                          |
| Workflows retain instance state/logs for a limited period                      | Maintain Dagflare's own durable event and artifact history                                                |
| The team requires no central Dagflare service                                  | Identity, scheduling, Git integration, state, and secrets must be independently deployable per account    |

The full evidence and qualifications are in the [Basin assessment](cloudflare-basin.md), [architecture](../design/architecture.md), and [validation plan](../design/validation-plan.md).

## Recommended direction

Build a Cloudflare-native control application around a stable asset graph. Use a Worker for UI/API, D1 for searchable metadata and run projections, R2 for definition snapshots and artifacts, Workflows for durable execution coordination, and a Durable Object for admission and scheduling coordination where necessary. Use Basin for analytics data. A container runner is a proposed adapter for Python/dbt workloads inside the same team's account.

This is a recommendation, not an implemented system. The highest-risk dependency is a repeatable dbt write path into Basin Catalog. The next proof should exercise table creation/replacement, model dependencies, test reporting, failures, and retry behavior against real Basin resources.

An alternative is to deploy Dagster OSS on Cloudflare Containers and add Basin integrations. That would reuse a mature engine, but its webserver, daemon, code-location services, and persistent instance storage add deployment requirements that have not been validated on Cloudflare. A thin catalog UI is easier but does not satisfy the orchestration and Git review goal. [DG18](sources.md)

## Evidence quality and limitations

- Sources are vendor-maintained GitHub repositories at pinned commits. Exact permalinks are in the [source register](sources.md). This makes the research reproducible even as live docs change.
- Direct vendor documentation websites were blocked by the cloud environment's egress policy during collection. Source documents were retrieved through the existing GitHub route. Live-page parity has not been verified.
- Documentation establishes supported interfaces and described behavior. It does not establish account entitlement, actual production throughput, undocumented API behavior, or successful integration.
- Local environment checks demonstrate Worker/D1/R2/Workflow execution and local dbt builds. They do not validate remote Basin, Access, Cloudflare Containers, or production deployment.
- Price tables are a dated snapshot, not a quote. Account-wide allowances, storage, operation fees, and optional products can change the total.
- Conflicting details in the retrieved docs are retained as questions. Examples include SQL credential scopes and Workflow concurrency figures. The design avoids relying on the disputed values.

## What to build first

The first vertical slice is: ingest a small event dataset, discover its Iceberg table, declare one derived asset, run a write-capable transformation, record the committed snapshot and checks, and show the resulting graph and run history. A PR then changes the definition, shows the asset impact, and promotes the reviewed version without writing to production from the preview.

The [product proposal](../design/product.md) and [technical gates](../design/validation-plan.md) define acceptance criteria. Building every Dagster or Prefect feature is outside the initial scope.
