# Technical validation plan and open questions

Status: required experiments before committing to the architecture. Local environment readiness and live product readiness are separate. The first application implementation should be a narrow vertical slice.

## What is already verified locally

- Wrangler 4.150.0 runs under the available Node 24 runtime and exposes `basin sql`, `basin catalog`, and `basin pipelines` commands.
- A local Worker launches a Workflow; the Workflow persists a known result to D1 and R2; the API retrieves and verifies the completed result.
- dbt-core 1.12.5 with dbt-duckdb 1.11.0 and DuckDB 1.5.6 builds one seed and one derived model and passes three data tests.
- Docker CLI and a reachable Docker engine are available. A Cloudflare Container build/deployment has not been validated.

These checks support research, docs, and local development. They do not prove remote Basin operations, real-account authorization, dbt-to-Iceberg compatibility, or a deployed Dagflare application.

## Gate A: Basin discovery and least-privilege access

Use a dedicated test bucket/catalog in the project owner's account. Confirm account entitlements and record actual API/CLI versions. List catalogs/namespaces/tables, inspect schema and snapshot metadata, query with bounded SELECT, and collect Pipeline/Catalog metrics.

Test separate catalog/storage reader and writer permissions. Resolve the apparent mismatch between the SQL guide's write-oriented examples and the Catalog guide's read-only guidance. Verify rotation and revoked-token behavior. Confirm catalog URI, warehouse mapping, query host, and account-specific storage hosts without putting secrets in logs.

Pass when a reader cannot write objects or table metadata, the application can explain insufficient permissions, and discovery/query observations have reproducible evidence. Missing public API coverage is a blocker to the affected feature, even if a dashboard can perform it.

## Gate B: Data-plane ingest and visibility

Create one structured stream, one pipeline, and one Catalog sink under test ownership. Submit valid events and malformed-schema events with explicit producer IDs. Measure acceptance, dropped-event metrics, sink activity, snapshot arrival, and query visibility over at least two roll intervals.

Verify the documented inability to modify schemas and attach a new sink to an existing table. Establish retention/replay semantics and supported replacement actions before designing automated migration. Test producer retries and document exactly where duplicate suppression applies.

Pass when the UI model can distinguish accepted, dropped, delivered, and queryable states and when cutover limitations are known. Do not promise lossless historical replay without a demonstrated source-of-truth and replay path.

## Gate C: dbt writes to Basin Catalog

Start with DuckDB's documented Iceberg REST catalog attachment. Prove creating and reading a test table. Then use dbt-duckdb against the same target with two dependent models, one table replacement, one supported incremental case, seeds/tests, and manifest/run-result ingestion.

Exercise a failing test, a failing SQL statement, a process interruption after commit, concurrent writes, catalog commit conflicts, and retry reconciliation. Evaluate temporary tables, rename/drop behavior, partitioned-table DELETE limitations, and compatibility with compaction.

Pass only for specific materializations whose outputs and snapshots are verified. If table replacement or incremental semantics fail, restrict the adapter explicitly or evaluate Spark/Trino; do not label generic dbt execution supported because a local DuckDB build passes.

## Gate D: Account-local execution runner

Build a pinned Python/dbt/DuckDB runner image, execute it locally through Docker, then deploy a Cloudflare Container in the same team account. Verify startup, memory/disk/CPU limits, image updates, authentication routes, timeout, cancellation, logs, artifacts, and result callbacks.

Determine whether the default scheduling policy is sufficient. Use the beta Durable Object scheduling policy only if runtime image selection is required. Snapshot capability is not mandatory; durable run artifacts must survive independently.

Pass when a runner interruption yields an honest unknown/reconciled outcome rather than duplicated writes and when production credentials cannot reach an untrusted preview job.

## Gate E: Orchestration and admission correctness

Create a small DAG with parallel independent nodes and a downstream node requiring both outputs and passing checks. Trigger duplicate requests and duplicate/out-of-order callbacks; interrupt between request persistence, Workflow creation, adapter submission, output commit, and event projection.

Test writer leases/fencing or conservative termination handling, repeated schedules, missed ticks, paused schedules, partial multi-model success, cancellation after commit, and bounded backfill admission. Retain the exact definitions and inputs used by in-flight runs during promotion.

Pass when logical runs are deduplicated, externally committed outputs are not lost or falsely rerun, and every blocked/skipped/error state has a persisted reason. Queues and Workflows do not automatically supply these application invariants.

## Gate F: Review and isolation

Open a test PR changing SQL, dependencies, checks, and a schema. Produce a deterministic artifact and graph diff at the exact head SHA. Ensure fresh pushes invalidate stale approval evidence. Use plan-only validation by default; test one bounded writable sandbox only with isolated data/resources and scoped credentials.

Pass when forked/untrusted PR code cannot access production and when merging/promoting does not enable an operationally paused schedule or reset run history.

## Gate G: Identity, operations, and portability

Deploy behind the team's Access application. Test valid JWTs, missing tokens, wrong issuer/audience, expired tokens, key rotation, alternative Worker routes, artifact authorization, role enforcement, and service identities.

Back up and restore application metadata/artifacts; test an upgrade and incompatible migration failure. Independently install into a second Cloudflare account and confirm that it uses only its own state/credentials/resources. Test that removing the application leaves existing analytical data intact.

Pass when the second installation operates without a required call to the first installation or a Dagflare-hosted endpoint.

## Questions requiring evidence

| Question                                                                             | Why it matters                                   | How to resolve                                                    |
| ------------------------------------------------------------------------------------ | ------------------------------------------------ | ----------------------------------------------------------------- |
| Which management/maintenance endpoints are public and stable?                        | Dashboard features do not imply integrable APIs  | Inventory API contracts and test read-only discovery first        |
| Can Basin SQL use fully read-only Catalog and R2 scopes?                             | Viewer/query permissions                         | Least-privilege account test                                      |
| What is the precise stream replay/retention contract?                                | Backfills and schema cutovers                    | Product docs/API test; vendor clarification if undocumented       |
| What can a new sink target during migration?                                         | Existing-table restriction                       | Supported resource replacement experiment                         |
| Does dbt-duckdb table/incremental behavior fit Basin?                                | Primary transformation path                      | Gate C with artifacts and snapshots                               |
| Which snapshot/time-travel/branch operations are supported by Basin SQL and Catalog? | Preview and recovery semantics                   | Query/catalog capability tests; no assumption from Iceberg alone  |
| How can remote queries be cancelled and their usage attributed?                      | User control and cost reporting                  | Inspect actual response/status APIs                               |
| Can runner auth use the intended secret route?                                       | Proxy credentials may not be raw process secrets | Gate D using supported runtime bindings                           |
| Which Workflow concurrency limit applies to this account?                            | Retrieved table/prose disagree                   | Account entitlement/API evidence; conservative admission          |
| What constitutes source freshness rather than maintenance activity?                  | Avoid misleading healthy status                  | Event-time watermarks and classified observations                 |
| Is a required jurisdiction supported?                                                | Possible adoption blocker                        | Confirm the current non-default-jurisdiction restriction          |
| What runner/metadata cost and latency are acceptable?                                | Product SLOs and pricing                         | Representative measured workload, including scans and maintenance |

## Sequence of implementation

First prove A–C on test resources. Then prove D–E with one adapter and a minimal API. Add the catalog/graph/run UI on top of the actual event model. Prove F–G before presenting the tool as independently deployable or safe for production team use.

Column-level lineage, broader adapters, dynamic partitions, sophisticated policy evaluation, and rich cost attribution follow the verified first slice. Customer workflow interviews can refine prioritization in parallel with these experiments.
