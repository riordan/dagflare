# Dagflare

Dagflare is a proposed asset-oriented data management application for Cloudflare Basin. Each team deploys the entire application into its own Cloudflare account. There is no shared Dagflare control plane, customer registry, hosted scheduler, or central credential service.

The goal is to make data assets understandable and operable: discover tables, follow lineage, inspect freshness and checks, run transformations, manage schedules, and review changes through Git pull or merge requests.

This repository currently contains research and design proposals. It does not yet contain a deployed application or a production-ready orchestrator.

## Start here

1. [Research summary and evidence boundaries](docs/research/README.md)
2. [Cloudflare Basin capability assessment](docs/research/cloudflare-basin.md)
3. [Dagster and Prefect product assessment](docs/research/dagster-prefect.md)
4. [Product scope and user workflows](docs/design/product.md)
5. [Architecture and execution model](docs/design/architecture.md)
6. [Git review, security, and deployment](docs/design/gitflow-security.md)
7. [Technical validation plan and unresolved questions](docs/design/validation-plan.md)
8. [Development environment and verified capabilities](docs/development.md)
9. [Primary sources and pinned revisions](docs/research/sources.md)

## Design commitments

- All persistent application state, orchestration, secrets, and data live in the team's account.
- Basin manages ingestion, Iceberg tables, and analytical queries. Dagflare adds lifecycle coordination and an asset-centric interface.
- Git is the source of truth for asset definitions. Runtime observations and operational controls have separate histories.
- A successful ingestion request is not proof of a successful table materialization.
- Basin SQL is read-only. Batch transformations and dbt require a write-capable execution engine.
- The first prototype targets one team; portability to another account is an explicit acceptance criterion.

Research was collected on October 13, 2026 from official source repositories. Vendor facts, design recommendations, and unverified integration hypotheses are distinguished in the documents.
