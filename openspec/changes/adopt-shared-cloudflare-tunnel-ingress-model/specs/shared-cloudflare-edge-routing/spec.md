## ADDED Requirements

### Requirement: Shared Cloudflare tunnel forwards ingress traffic to cluster ingress controller
The platform MUST run a shared Cloudflare tunnel workload that terminates Cloudflare Tunnel connectivity and forwards public HTTP(S) requests to the cluster ingress controller service.

#### Scenario: Shared tunnel forwards traffic to ingress controller
- **WHEN** a request for a configured public hostname arrives through Cloudflare Tunnel
- **THEN** the shared `cloudflared` workload forwards the request to the ingress controller service endpoint in-cluster

### Requirement: Applications own hostname routing through app-local Ingress resources
Public application hostname routing MUST be defined in each application's own `Ingress` manifests, and the shared tunnel layer MUST NOT require per-app direct service host routing as the primary model.

#### Scenario: App exposes hostname through its own ingress manifest
- **WHEN** an application defines a valid `Ingress` rule for `app.jellylabs.xyz`
- **THEN** traffic for that hostname is routed by the ingress controller to the application's Kubernetes Service without adding an app-specific cloudflared service route

### Requirement: Public hostname records map explicitly to the shared tunnel target
Each public app hostname MUST have an explicit DNS record in Cloudflare that maps to the shared tunnel target for deterministic routing and governance.

#### Scenario: Linkding and Streamlit hostnames share one tunnel target
- **WHEN** DNS is configured for `linkding.jellylabs.xyz` and `streamlit.jellylabs.xyz`
- **THEN** both records resolve to the same tunnel target while remaining distinguishable by hostname in ingress routing

### Requirement: Migration from app-local tunnel to shared edge is reversible
The migration process MUST preserve service continuity for existing hostnames and include a documented rollback path until cutover validation succeeds.

#### Scenario: Cutover validation fails
- **WHEN** post-cutover checks fail for an existing public hostname
- **THEN** operators can restore the previous app-local tunnel path using documented rollback steps
