## Why

Homepage is currently a brittle hand-maintained dashboard whose service state often diverges from the actual cluster because labels, selectors, and widget assumptions are inconsistent. This change is needed now to make Homepage reliable for day-to-day use and to establish a repo convention that keeps the dashboard current as new apps are added.

## What Changes

- Rework Homepage into a service catalog that cleanly separates ingress-backed applications, curated non-ingress applications, and curated platform/operator entries.
- Standardize application identity metadata so Homepage status checks can rely on consistent Kubernetes labels instead of ad hoc selectors.
- Define how Homepage metadata is declared for applications that belong on the dashboard, preferring ingress annotations when an app is ingress-backed.
- Treat local-only web applications as valid ingress-backed Homepage entries when they are meant to be launched from the dashboard.
- Remove or replace misleading Homepage tiles and widgets that do not reflect real cluster state, including broken operator and service representations.
- Establish an ongoing rule that new deployments must include an explicit Homepage presence decision so the dashboard stays aligned with the cluster over time.

## Capabilities

### New Capabilities
- `homepage-service-catalog`: Defines how applications and operator-facing services are represented in Homepage, how metadata is declared, and how new deployments stay aligned with the dashboard.

### Modified Capabilities

None.

## Impact

- Affected code and config: `apps/base/homepage/`, application manifests under `apps/base/` and `apps/production/`, and any ingress resources that should carry Homepage metadata.
- Affected systems: Homepage, Traefik ingress routing, application labeling conventions, and operator-facing services such as Flux and CloudNative-PG.
- Process impact: future app changes will need to include Homepage annotations or curated entries when the app should appear on the dashboard.
