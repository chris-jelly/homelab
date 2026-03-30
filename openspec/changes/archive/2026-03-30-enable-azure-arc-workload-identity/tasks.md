## 1. Manual operator tasks

- [x] 1.1 Register the required Azure resource providers for `rg-jellyhomelab` in `Canada Central`: `Microsoft.Kubernetes`, `Microsoft.KubernetesConfiguration`, and `Microsoft.ExtendedLocation`.
- [x] 1.2 Connect the production cluster to Azure Arc in `rg-jellyhomelab` and `Canada Central` with Azure CLI `connectedk8s` tooling.
- [x] 1.3 Verify the Arc-connected cluster resource exists and the `azure-arc` agents are running in the cluster.

## 2. Manual operator tasks

- [x] 2.1 Enable Arc OIDC issuer support and Arc workload identity support for the connected cluster.
- [x] 2.2 Retrieve and document the Arc-managed OIDC issuer URL.
- [x] 2.3 Update the k3s control-plane configuration so `service-account-issuer` uses the Arc-managed issuer URL.
- [x] 2.4 Restart k3s and document the restart and rollback procedure.

## 3. Manual operator tasks

- [x] 3.1 Validate the Arc-managed issuer URL is reachable and returns the expected OIDC discovery document.
- [x] 3.2 Validate the issuer's JWKS endpoint is reachable and suitable for Microsoft Entra federation.
- [x] 3.3 Mint a fresh projected service account token and verify its `iss` claim matches the Arc-managed issuer URL.

## 4. Agent and repo tasks

- [x] 4.1 Update `specs/001-cluster-bootstrap.md` with Arc prerequisites, provider registration, cluster connection in `Canada Central`, issuer retrieval, k3s reconfiguration, validation steps, and cleanup commands.
- [x] 4.2 Start replacing the current manual Azure credentials bootstrap step in `specs/001-cluster-bootstrap.md` with workload identity-based bootstrap guidance where Arc support is now available.
- [x] 4.3 Update the Mealie backup change to consume the Arc-managed `kubernetes_oidc_issuer_url` and Arc workload identity annotations and labels.
- [x] 4.4 Document the standard service account, pod label, and federated credential pattern that future Azure-integrated workloads should follow.
