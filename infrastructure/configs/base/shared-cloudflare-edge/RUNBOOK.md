# Shared Cloudflare Edge Migration Notes

## DNS cutover (operational)

Update Cloudflare DNS records to point both hostnames at the shared tunnel target.

- `linkding.jellylabs.xyz` -> shared tunnel target
- `streamlit.jellylabs.xyz` -> shared tunnel target

This repository does not manage Cloudflare DNS records yet. Apply DNS changes manually in Cloudflare.

## Validation checklist

After Flux reconciliation and DNS propagation:

1. `curl -I https://linkding.jellylabs.xyz`
2. `curl -I https://streamlit.jellylabs.xyz`
3. Confirm both return expected HTTP status and app responses.
4. Confirm `kubectl get deploy -n linkding cloudflared` returns not found.

## Rollback

If cutover validation fails:

1. Restore `apps/production/linkding/cloudflare.yaml` in `apps/production/linkding/kustomization.yaml`.
2. Re-add `linkding-cloudflare-tunnel-credentials` mapping in `apps/production/linkding/external-secrets.yaml`.
3. Revert DNS for `linkding.jellylabs.xyz` to the previous app-local tunnel target if needed.
4. Reconcile Flux and re-run `curl` checks.
