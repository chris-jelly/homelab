## 1. Shared Edge Tunnel Foundation

- [x] 1.1 Create a shared edge directory under `infrastructure/configs/base/` with a `kustomization.yaml` and include it from the infrastructure config kustomization tree
- [x] 1.2 Add shared `cloudflared` Deployment and ConfigMap manifests that forward traffic to Traefik instead of app-specific services
- [x] 1.3 Add an ExternalSecret and Secret key mapping for shared tunnel credentials sourced from Azure Key Vault
- [x] 1.4 Ensure shared cloudflared workload has production-ready probes, resource requests/limits, and replica settings

## 2. App Routing Ownership via Ingress

- [x] 2.1 Add or update Linkding ingress manifest to own `linkding.jellylabs.xyz` routing through Traefik
- [x] 2.2 Add or update `sales-pipeline-pulse` ingress manifest (namespace `streamlit`) to own `streamlit.jellylabs.xyz` routing through Traefik
- [x] 2.3 Verify app service backends and ports referenced by ingress rules match deployed Service resources

## 3. Migration and Cleanup

- [x] 3.1 Create migration notes documenting operational DNS record updates that must point hostnames to the shared tunnel target (runbook-managed, not repo-managed)
- [x] 3.2 Remove Linkding app-local tunnel manifests from its production kustomization after shared routing is ready
- [x] 3.3 Remove obsolete Linkding tunnel credential secret mappings no longer used after cutover
- [x] 3.4 Document rollback steps for temporarily restoring Linkding app-local tunnel routing if validation fails

## 4. Validation

- [x] 4.1 Run `kustomize build` (or equivalent Flux render validation) for both `infrastructure/configs/base` and `apps/production`, and resolve manifest errors
- [x] 4.2 Validate hostname routing for `linkding.jellylabs.xyz` and `streamlit.jellylabs.xyz` after deployment
- [x] 4.3 Confirm no remaining app-local cloudflared workloads are active for migrated applications
