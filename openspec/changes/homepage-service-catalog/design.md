## Overview

This change turns Homepage from a hand-curated list of services into a cluster-aligned service catalog. The design separates three representation modes: discovered ingress-backed applications, curated non-ingress applications, and curated platform/operator entries. It also introduces a repo convention that Homepage presence is part of application definition rather than an afterthought.

## Goals

- Make Homepage status indicators reflect real cluster state.
- Keep Homepage aligned with both existing and future applications.
- Prefer metadata that lives with the application resource instead of a central dashboard file.
- Support both public and local-only applications.
- Avoid requiring every workload in the cluster to appear as a Homepage tile.

## Non-Goals

- Expose every application publicly on the internet.
- Create a Homepage tile for Homepage itself.
- Represent every internal component of multi-part systems such as Airflow.
- Replace platform observability with Homepage alone.

## Service Catalog Model

Homepage entries fall into three categories:

1. Ingress-backed applications
   - Applications with an ingress, whether public or local-only.
   - Preferred representation for app tiles.
   - Homepage metadata should live on the ingress using `gethomepage.dev/*` annotations.

2. Curated non-ingress applications
   - Applications that belong on Homepage but do not have an ingress.
   - Represented manually in Homepage configuration.
   - Used when an app has no HTTP entrypoint or when adding an ingress is not worth the complexity.

3. Curated platform and operator services
   - Operator-facing services such as Flux, CloudNative-PG, cert-manager, and external-secrets.
   - Represented manually because discovery and generic app widgets are often misleading.
   - May use pod selectors, Prometheus metrics, or documentation links instead of native Homepage widgets.

Excluded workloads are allowed. Jobs, helper controllers, and backing components that are not operator entrypoints do not need Homepage representation.

## Metadata Model

The design separates workload identity from Homepage presentation.

### Kubernetes Identity Labels

Applications that belong in Homepage should use standard Kubernetes app labels where they make sense:

- `app.kubernetes.io/name`
- `app.kubernetes.io/instance`
- `app.kubernetes.io/part-of`
- `app.kubernetes.io/component` for multi-component systems

Homepage status checks should rely on these labels or explicit selectors, not on legacy `app:` labels alone.

### Homepage Presentation Metadata

Homepage presentation metadata should be declared as close as possible to the user access path:

- Ingress-backed apps: `gethomepage.dev/*` annotations on the ingress
- Non-ingress apps: curated manual entry in Homepage config
- Platform services: curated manual entry in Homepage config

This keeps ownership local to the app definition while avoiding duplicate central catalog entries.

## Ingress Strategy

Ingress is treated as an HTTP routing mechanism, not as a synonym for public internet exposure.

- Public apps may continue using public hostnames.
- Local-only apps may use private DNS and private ingress routes.
- If a web app belongs in Homepage and an ingress is practical, a local-only ingress is the preferred path.

This allows internal and external apps to follow the same Homepage discovery model.

## Existing App Handling

Existing apps are part of the migration baseline rather than exceptions.

- Normalize labels for current Homepage-worthy apps.
- Add Homepage annotations to current ingress-backed apps.
- Reclassify broken manual tiles based on whether they should be discovered, curated, or removed.

Current direction for known apps:

- `homepage`: no app tile
- `linkding`: ingress-backed and discovered
- `sales-pipeline-pulse`: ingress-backed and discovered
- `airflow`: represent only the webserver entrypoint
- `actualbudget` and `audiobookshelf`: decide whether to add local or public ingress; otherwise keep curated
- `grafana` and `prometheus`: curated observability entries
- `flux`, `cloudnative-pg`, `cert-manager`, `external-secrets`: curated platform entries

## Homepage Layout Direction

The dashboard should be reorganized around intent:

- Top-level cluster information widgets for cluster and node state
- Applications section for ingress-backed and curated app entries
- Platform section for operators and cluster services
- Observability section for Grafana, Prometheus, and related tools

Homepage itself should remain the dashboard page and should not also appear as an app tile.

## Widget Strategy

- Remove or replace widgets that have no working implementation in the deployed Homepage image.
- Prefer generic status, site monitoring, or Prometheus-backed metrics over misleading native widgets.
- Use explicit pod selectors where app identity cannot be inferred safely from standard labels.

This is especially important for Flux and CloudNative-PG, whose current Homepage representations do not match the cluster.

## Governance and Review

New app changes should include an explicit Homepage presence decision.

- If the app belongs in Homepage and has an ingress, the ingress should carry Homepage annotations.
- If the app belongs in Homepage but has no ingress, the change should add or update a curated Homepage entry.
- If the app does not belong in Homepage, the change should intentionally leave it out.

This turns Homepage alignment into a normal review concern and prevents dashboard drift.

## Risks and Mitigations

- Risk: duplicate entries from discovery plus manual config.
  - Mitigation: remove manual entries when an app becomes ingress-discovered.

- Risk: inconsistent labels keep status badges unreliable.
  - Mitigation: normalize labels as part of the migration.

- Risk: local-only DNS and ingress conventions remain ambiguous.
  - Mitigation: choose a local hostname pattern during implementation and apply it consistently.

- Risk: multi-component systems are modeled too broadly.
  - Mitigation: represent operator-facing entrypoints only, such as the Airflow webserver.
