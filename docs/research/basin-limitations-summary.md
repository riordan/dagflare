# Basin limitations: short summary

As of October 14, 2026, based on the verified official documentation. The [detailed assessment](basin-limitations-2026-10-14.md) contains sources and qualifications.

**Basin separates ingestion writes from analytical queries.** Pipelines transforms incoming streams and delivers records to sinks. Basin SQL queries stored Iceberg tables but is read-only. Coming from BigQuery or Snowflake, the key adjustment is that computing a SQL result and persisting a derived table require separate execution paths.

## Pipelines

- Supports stateless filtering, reshaping, calculated fields, and routing from streams to sinks. General stateful joins, aggregates, and windows are not established current capabilities.
- No documented native upsert, merge, row-delete, or CDC-application contract. A delete event is data until another mechanism interprets it.
- Stream schemas, pipeline SQL, and sink configurations are immutable after creation; changes need replacement and cutover planning.
- A new Catalog sink cannot attach to a pre-existing Iceberg table.
- Ingestion acknowledgement does not establish query visibility or completeness. Invalid-schema events can be accepted then dropped. Catalog rolling defaults to 300 seconds, with a documented minimum interval of 60 seconds; this is not an end-to-end latency guarantee.
- Default quotas: 20 streams, 20 pipelines, 20 sinks per account; 5 MB per request. Historical replay and application-event deduplication contracts need verification.

## SQL

- Supports substantial read queries, including joins, CTEs, aggregates, and inline windows, but no INSERT/UPDATE/DELETE or CREATE/DROP/ALTER. It cannot persist a model through CREATE TABLE AS SELECT.
- Queries Parquet data in Catalog-managed Iceberg tables; raw CSV/JSON files are outside this query interface.
- Dialect gaps include OFFSET, UNNEST, PIVOT, lateral derived tables, and named WINDOW clauses. Default LIMIT is documented as 500, maximum LIMIT as 10,000.
- Expensive joins, aggregates, and windows can exceed memory budgets or time out. Supported syntax does not guarantee arbitrary-scale execution.

## Design implications

Mutable tables remain possible through compatible Iceberg writers committing through Basin Catalog, subject to engine-specific support. Pipeline insertion and Basin SQL querying do not themselves provide that mutation path.

**Dagflare** should own declarative assets, dependencies, evidence, health, and safe changes, with distinct contracts for streaming and batch execution. **dbt-cloudflare-basin** should be independently usable and select a proven write-capable executor for conventional dbt materializations.

Append-only change events plus tombstones can represent logical current state, but require correct ordering, deduplication, retention, and query performance. They do not physically erase historical data.
