## 1. Baseline the current Homepage inventory

- [x] 1.1 Inventory all current Homepage entries and classify each as ingress-backed app, curated non-ingress app, curated platform service, or excluded workload
- [x] 1.2 Confirm the target representation for current apps including Linkding, Sales Pipeline Pulse, Airflow webserver, ActualBudget, Audiobookshelf, Grafana, Prometheus, Flux, CloudNative-PG, cert-manager, and external-secrets
- [x] 1.3 Decide the local-only ingress hostname convention for internal web apps that should appear in Homepage

## 2. Normalize application metadata

- [x] 2.1 Add or align standard Kubernetes application labels on Homepage-worthy workloads and services
- [x] 2.2 Update Homepage selectors or pod selectors so status checks resolve the intended workloads
- [x] 2.3 Limit multi-component systems to operator-facing entrypoints, including representing Airflow through the webserver only

## 3. Move ingress-backed apps to declared Homepage metadata

- [x] 3.1 Add `gethomepage.dev/*` annotations to current ingress-backed apps that belong in Homepage
- [x] 3.2 Remove duplicate manual Homepage entries for apps that become ingress-discovered
- [x] 3.3 Add local-only ingresses for internal web apps where ingress-backed Homepage discovery is preferred and practical

## 4. Rework curated Homepage configuration

- [x] 4.1 Remove the Homepage self-tile and reorganize Homepage sections around applications, platform services, and observability
- [x] 4.2 Replace broken or misleading widgets and service definitions for Flux, CloudNative-PG, and other non-app operators
- [x] 4.3 Keep curated entries only for non-ingress applications and operator-facing services that should remain manual

## 5. Encode the governance rule in the repo

- [x] 5.1 Update specs and supporting docs so Homepage presence is part of the deployment convention for Homepage-worthy apps
- [x] 5.2 Document the preferred decision path: ingress annotations for ingress-backed apps, curated entries for non-ingress apps, and intentional exclusion for non-dashboard workloads
- [x] 5.3 Add review guidance so new app changes include an explicit Homepage presence decision in the same change

## 6. Validate the new catalog behavior

- [x] 6.1 Verify Homepage renders the expected application, platform, and observability sections without duplicate entries
- [x] 6.2 Verify status indicators and links are accurate for migrated existing apps
- [x] 6.3 Verify new ingress-backed apps can be added through metadata on their own manifests without expanding the central Homepage catalog
