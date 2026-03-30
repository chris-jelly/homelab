# azure-arc-workload-identity Specification

## Purpose
TBD - created by archiving change enable-azure-arc-workload-identity. Update Purpose after archive.
## Requirements
### Requirement: Cluster can be connected to Azure Arc
The cluster SHALL have a documented and repeatable onboarding path to Azure Arc-enabled Kubernetes.

#### Scenario: Arc onboarding succeeds
- **WHEN** an operator follows the cluster onboarding procedure
- **THEN** the cluster is registered as an Azure Arc-enabled Kubernetes resource
- **AND** the Azure Arc agents are present and healthy in the cluster

### Requirement: Cluster exposes an Arc-managed issuer for workload identity
The cluster SHALL use an Azure Arc-managed OIDC issuer URL for Microsoft Entra workload identity federation.

#### Scenario: Arc-managed issuer is discoverable
- **WHEN** an operator retrieves the OIDC issuer URL for the connected cluster
- **THEN** the issuer URL is publicly reachable
- **AND** it serves `/.well-known/openid-configuration` and `/openid/v1/jwks`

### Requirement: Newly minted service account tokens use the Arc issuer
The cluster SHALL mint new projected service account tokens with the Azure Arc-managed issuer URL.

#### Scenario: Fresh token uses Arc-managed issuer
- **WHEN** a new projected service account token is minted after k3s is reconfigured
- **THEN** the token's `iss` claim equals the Arc-managed issuer URL

### Requirement: Arc workload identity mutation contract is available to workloads
The cluster SHALL provide the service account annotation and pod label contract needed for Azure workload identity-enabled pods.

#### Scenario: Workload identity-ready pod configuration is documented
- **WHEN** operators configure an Azure-integrated workload
- **THEN** the service account annotation for Azure client ID is documented
- **AND** the required pod label for workload identity mutation is documented

### Requirement: Arc integration has documented validation and cleanup
The cluster SHALL document how to validate and remove the Arc workload identity integration.

#### Scenario: Operators can validate and clean up Arc integration
- **WHEN** operators review the cluster runbook for Azure Arc workload identity
- **THEN** they can verify cluster connection, issuer reachability, token issuer claims, and Arc agents
- **AND** they can remove the Arc connection with the documented cleanup command if the integration must be rolled back

