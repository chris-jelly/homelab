## ADDED Requirements

### Requirement: Mealie runs as a production application in the homelab cluster
The cluster SHALL deploy Mealie as a production application through Flux-managed manifests in the `mealie` namespace.

#### Scenario: Production manifests include Mealie application resources
- **WHEN** the production apps Kustomization is rendered
- **THEN** it includes Mealie application resources for the `mealie` namespace
- **AND** those resources define a Kubernetes `Deployment` and `Service` for Mealie

### Requirement: Mealie persists application filesystem state
The Mealie application SHALL mount persistent storage at `/app/data` so application filesystem state survives pod restarts and rescheduling.

#### Scenario: Mealie pod mounts persistent app data storage
- **WHEN** the Mealie pod starts
- **THEN** it mounts a persistent volume at `/app/data`
- **AND** the volume is not backed by `emptyDir`

### Requirement: Mealie uses a CloudNativePG-managed PostgreSQL database
The Mealie application SHALL use PostgreSQL provided by a CloudNativePG cluster in the `mealie` namespace and SHALL not run with SQLite in production.

#### Scenario: Mealie is configured for CNPG PostgreSQL
- **WHEN** the Mealie deployment is rendered
- **THEN** it sets `DB_ENGINE` to `postgres`
- **AND** it provides PostgreSQL host, port, database, username, and password values from managed cluster resources or derived secrets

### Requirement: Mealie is reachable through the cluster ingress path
The cluster SHALL expose Mealie through Traefik ingress with TLS for its production hostname.

#### Scenario: Mealie ingress is configured for production access
- **WHEN** the production manifests are applied
- **THEN** the `mealie` namespace contains an ingress resource for the production Mealie hostname
- **AND** the ingress is configured to use TLS for that hostname

### Requirement: Mealie uses externalized secret material
The Mealie deployment SHALL obtain sensitive configuration through Kubernetes Secrets sourced by External Secrets or database-generated secrets, and SHALL not hardcode secrets in git-tracked manifests.

#### Scenario: Sensitive Mealie configuration is not embedded in manifests
- **WHEN** the Mealie deployment references sensitive values such as database credentials
- **THEN** those values are sourced from Kubernetes Secrets
- **AND** the manifests do not embed plaintext secret values

### Requirement: Mealie supports OpenAI integration through externalized configuration
The Mealie deployment SHALL support OpenAI integration with the API key sourced from Azure Key Vault through External Secrets rather than from git-tracked manifests.

#### Scenario: OpenAI integration is wired from secret material
- **WHEN** the Mealie deployment is rendered with OpenAI enabled
- **THEN** the required OpenAI API key is sourced from a Kubernetes Secret populated by External Secrets
- **AND** the manifests do not embed the OpenAI API key in plaintext

### Requirement: Mealie is discoverable in Homepage
The Mealie ingress SHALL include Homepage discovery metadata so the service appears through Homepage's ingress-backed app discovery model.

#### Scenario: Mealie ingress exposes Homepage metadata
- **WHEN** the production Mealie ingress is rendered
- **THEN** it includes `gethomepage.dev/*` annotations needed for Homepage discovery
- **AND** Mealie does not require a separate curated Homepage entry solely for basic app discoverability
