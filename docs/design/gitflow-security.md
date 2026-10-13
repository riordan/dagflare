# Git review, security, and deployment

Status: proposed. This document defines requirements; it does not claim that Git integration, RBAC, or deployment automation already exists.

## Git as the definition source

The team owns its definition repository. SQL, dbt projects, manifests, checks, partition mappings, and policy definitions are reviewed through its normal Git provider. GitHub is the first integration; GitLab MR support can follow using the same provider-neutral definition/change model.

The UI can draft a change and open a PR/MR, but it does not silently mutate the active production definition. Editing an owner or description in an authoritative definition also creates a reviewable change. Runtime annotations can be separate only when visibly identified as such.

The first install can use CI-signed bundles pushed to the account-owned API, avoiding a mandatory Git App. A richer integration can use a GitHub App owned and configured by each team, with installation permissions scoped to its repositories. Dagflare must not depend on a central developer-owned GitHub App or a shared webhook/credential broker. Public release distribution remains independent of customer runtime access.

## Review and promotion lifecycle

1. A PR changes definitions or transformation code.
2. Unprivileged CI validates the schema and graph and builds a deterministic artifact at the PR head SHA.
3. Dagflare compares it with the recorded base definition/SHA and displays changed assets, dependencies, target schemas, checks, policies, and downstream impact.
4. Static validation and fixture-based execution produce status evidence tied to that exact head SHA.
5. An authorized operator may explicitly request a bounded writable sandbox using separate resources and credentials.
6. Merge creates a deployable bundle at the reviewed merge SHA. Production promotion requires current authorization and successful validation of that artifact.
7. Dagflare records the active definition change. Operational pause/admission settings persist; schedules are enabled by explicit installation policy or operator action.

Record repository identity, provider delivery ID, PR/MR ID, base/head/merge SHAs, deployed definition digest, compiler version, CI identity, and validation artifact hashes. Reject stale approval when the head artifact changes. The graph diff should identify which comparison base was used, rather than mixing repository and currently deployed state.

## Preview environments and data isolation

A UI/code preview is different from a writable data preview. Default reviews use fixtures, generated sample data, and plans. Do not clone a production warehouse or give forked PRs production secrets.

For a writable sandbox, assign distinct resource IDs, catalogs/buckets or namespace constraints with proven permissions, writer credentials, and a resource ownership label. Confirm that the permissions actually isolate object access: a different namespace name alone is not a security boundary. The current Catalog token/vended-credential behavior makes storage rights particularly important. [Catalog authentication evidence](../research/sources.md)

Budget capacity against account-wide stream/sink/pipeline limits. Reserve a small number of shared test slots, expire sandbox leases, and clean up only resources that Dagflare created and still owns. Plan-only previews require no new Basin streams. A stream schema replacement is not a trivial update; show producer cutover, drain, retention, and rollback impact.

Use separate least-privilege readers when optional production samples are allowed. Sample exports can contain personal data; make their purpose and retention explicit. A branch environment may read approved fixtures without seeing production data.

Do not infer Iceberg branch/tag support, zero-copy cloning, or transactional multi-table promotion from Iceberg's general format capabilities. These must be established for the actual Basin APIs and chosen engine. Early writable previews should use simple independently managed test resources.

## Identity and authorization

Cloudflare Access provides team authentication. The Worker verifies the JWT signature, configured issuer, audience, expiry, and approved identity source, using the team's rotating public keys. It rejects spoofed identity headers and unauthenticated requests even if an alternate Worker route reaches the API. Restrict or protect workers.dev and preview routes when deploying on an Access-protected custom domain. [Access validation evidence](../research/sources.md)

Authentication establishes who the caller is; Dagflare authorization determines what the caller can do. Apply authorization to every API, artifact download, query preview, runner submission, and integration-management action.

| Role             | Initial allowed actions                                                                        |
| ---------------- | ---------------------------------------------------------------------------------------------- |
| Viewer           | Read permitted catalog, lineage, run metadata and approved previews                            |
| Operator         | Viewer actions; execute approved definitions, retry/cancel, pause schedules                    |
| Editor           | Viewer actions; propose and validate definition changes; no implicit production promotion      |
| Admin            | Configure integrations/roles/resource scope and authorize promotion; sensitive actions audited |
| Service identity | Only declared CI, runner, or webhook operations within explicit scope                          |

Roles can be combined deliberately; Editor should not silently inherit unrestricted runtime writer access. Git permissions and app roles are separate. A Git contributor must not automatically receive Admin privileges, and app administrators must not bypass protected-branch rules.

A Viewer may need metadata access but no row-level sample access. Treat previews and raw logs/artifacts as separate permissions because they can expose sensitive data. App restrictions are ineffective if the user can retrieve a broader backend token; never expose those tokens to the browser.

## Credentials and network boundaries

Use native bindings for D1, R2 application storage, Workflows, and account-local resources where supported. Management-plane APIs and external Iceberg clients may still need tokens. Store them in the team's Worker secrets or approved team-owned secret facility, referenced by environment and adapter. This research did not establish a universal token-free Basin administration binding.

Separate provisioner, catalog reader, data writer, query reader, and CI identities. Scope both catalog and object-storage permissions. Rotate credentials without rewriting historical definition bundles. Never put credential values into Git, dbt artifacts, SQL logs, URL parameters, container images, or audit payloads.

Basin SQL documentation and Catalog documentation currently differ in their token examples. Test genuine read-only storage/catalog/query scopes and preserve the actual successful scope as an installation contract. Do not silently request account-wide admin access as the permanent solution.

Outbound destinations are explicit integration requirements. Management uses Cloudflare API endpoints, queries use the documented SQL host, and catalog/storage use account-specific Cloudflare Storage hosts. Git API access has a different authorization scope from Git transport. Document needed hosts for the team's deployment and avoid a wildcard egress policy merely for convenience.

Cloudflare onboarding proxy-backed secret bindings are not automatically raw credentials for a local container process. The live integration proof must verify the supported authentication route before assuming that a placeholder token works in DuckDB's native extension or a Docker build.

## Untrusted code and event handling

Git webhooks require provider-signature verification, delivery-ID deduplication, an explicit allowed repository, and replay handling. CI bundle submissions require an authenticated service identity and artifact digest verification; do not trust repository names or SHAs merely because they appear in a request body.

Untrusted PR code has no production credentials. Avoid privileged CI patterns that execute a contributor's scripts in a context with writer secrets. A container running arbitrary SQL/Python still needs resource limits, allowed destinations, scoped data access, and clean attempt-specific state. Sharing an account does not make all code equally trusted.

Runner callbacks bind to the logical run/attempt and authenticated executor. Reject callbacks from obsolete attempts and use sequence/version checks so late “running” events cannot overwrite a terminal state. Duplicate events should produce the same projection.

## Rollback and audit

Rollback has at least three separate operations:

- **Application rollback:** change the Worker/container software version, subject to metadata schema and image compatibility.
- **Definition rollback:** promote a prior asset definition artifact for future executions.
- **Data recovery:** restore/recompute table state through a verified engine and retained data snapshots.

None implies the others. A prior definition can still be incompatible with an evolved source schema. Snapshot expiration can remove the data necessary for recovery. A failed migration therefore needs a specific recovery plan rather than a single generic rollback button.

Audit actor identity, action, authorization scope, target, definition/environment, request/delivery ID, and result. Retain operation history independently of Workflow's built-in log retention. Secret reads should be minimized and never produce secret values in audit records. Export/retention policies belong to the team; audit records are not automatically a compliance certification.

## Independent deployment and upgrades

The install plan creates application metadata/artifact resources and integrates existing analytical resources by reference. It must distinguish resources it owns from data the team already owned. Uninstall removes the application by default and does not drop analytical tables or buckets.

Pin application release, runner image digest, manifest schema, and migration versions. Validate migrations before activation. Perform a second-account deployment using separate identity, Git/CI credentials, and test data; prove no runtime calls depend on the first account.

No required central telemetry, billing, license server, or global account registry is part of this design. Optional update checks and support exports are opt-in and must function without transferring team data by default.
