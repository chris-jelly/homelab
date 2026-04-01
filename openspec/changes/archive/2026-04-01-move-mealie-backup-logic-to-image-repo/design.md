## Context

The Mealie file-backup CronJob now proves the key platform pieces: Azure Arc workload identity injection works, the published `ghcr.io/chris-jelly/mealie-backup-azcopy` image can authenticate to Blob Storage, and the Mealie PVC can be archived successfully. The remaining problem is ownership of backup behavior.

Today, `homelab` still carries `backup-script-configmap.yaml` with the entrypoint logic and inline Python for retention pruning, plus `backup-config-configmap.yaml` for runtime settings. That makes the cluster repo own implementation details that really belong in `homelab-images` or Azure storage policy. It also creates a brittle contract: if the image is missing `python3` or changes AzCopy behavior, the Kubernetes-side script fails at runtime.

This change formalizes a cleaner boundary. `homelab` should define the CronJob, schedule, service account, mounts, and a small set of environment variables. A small inline shell wrapper remains acceptable in the cluster repo for archive creation and upload, but retention cleanup should move out to Azure storage lifecycle management. Because the policy lives outside this repo, this change also needs durable handoff documents that tell operators and `jellylabs-dsc` exactly what contracts to satisfy.

## Goals / Non-Goals

**Goals:**
- Simplify the Mealie file-backup script so it only handles archive creation and upload.
- Reduce the Kubernetes manifests to deployment wiring and env configuration only.
- Remove the current prune complexity and Python dependency from the Kubernetes-side script.
- Provide handoff docs that can be used to implement the Azure lifecycle policy in the other repo and document the simplified local script contract.
- Keep the existing workload identity wiring, Blob target, and schedule intact during the refactor.

**Non-Goals:**
- Changing the CNPG backup path or the Mealie database backup design.
- Redesigning Azure workload identity itself or the Mealie managed identity setup.
- Generalizing the pattern for every backup workload in the cluster in the same change.
- Eliminating all shell scripting from `homelab`.

## Decisions

### Keep a small inline shell wrapper, but remove prune logic
The Mealie file-backup CronJob may continue to mount a small shell script from a ConfigMap as long as the script is limited to archive creation and AzCopy upload. Retention pruning will move out of the job and into Azure Blob lifecycle management.

Rationale:
- A short shell wrapper that is tightly scoped to one CronJob is operationally acceptable in this repo.
- The problematic part was the prune logic, AzCopy output parsing, and hidden `python3` dependency, not the existence of shell itself.
- Retention is not really application behavior here; it is storage lifecycle behavior scoped to the `mealie/files/` prefix.
- This removes the inline Python and prune logic without forcing a larger image-contract refactor.

Alternatives considered:
- Move even the small shell wrapper into the image: cleaner architectural separation, but no longer clearly worth the extra coordination.
- Keep prune logic in the image: workable, but unnecessary if Azure storage can enforce age-based cleanup directly.

### Keep the small runtime contract in Kubernetes, with minimal indirection
`homelab` will keep the Mealie backup runtime contract small. The CronJob and backup ConfigMaps may remain if they improve readability, but any configuration that exists only to support prune behavior should be removed.

Rationale:
- These values are workload configuration, not image implementation.
- With a tiny shell wrapper, a small amount of local configuration is still easy to reason about.
- The key simplification is removing prune-related complexity, not forcing every variable into one manifest.

Alternatives considered:
- Inline every env var directly into the CronJob: workable, but not required if a small ConfigMap remains clearer.
- Hardcode all values into the image: simpler image API, but couples the image too tightly to one workload instance.

### Publish handoff docs for the simplified local script and Azure retention policy
The change will add concise handoff docs under the Mealie app path that describe the simplified backup contract and the Azure lifecycle policy expected from Terraform.

Rationale:
- The simplified backup shape still benefits from a durable reference in the consuming repo.
- The Terraform side needs a clear statement of the target retention window and prefix scope.

Alternatives considered:
- Only document the contract in the OpenSpec change: useful for planning, but not ideal as a durable operational reference.

### Use Azure Blob lifecycle management for file-backup retention
Retention for `mealie/files/` will move to an Azure Blob lifecycle management policy managed from Terraform, most likely with a prefix-scoped rule on `sthomelabbackups` that deletes old blobs by age.

Rationale:
- The current retention goal is operationally close to "keep about eight weekly backups," which maps well to an age-based policy such as 56-60 days.
- Blob lifecycle policy is the right ownership layer for storage cleanup.
- It avoids embedding listing and prune logic into the backup job or the helper image.

Alternatives considered:
- Keep exact count-based pruning in the job: more precise, but more complex and harder to maintain.
- Add a separate cleanup CronJob: still more moving parts than a storage-native policy.

### Keep the current workload identity and storage contract unchanged
This refactor will not change `ServiceAccount/mealie-backup`, the workload identity labels, or the target `mealie/files` Blob prefix.

Rationale:
- The current issue is ownership of logic, not authentication design.
- Minimizing unrelated changes lowers rollout risk.

Alternatives considered:
- Combine this with broader Mealie backup auth cleanup: too much scope for one refactor.

## Risks / Trade-offs

- [Script cleanup and Azure policy rollout can drift] -> Write the handoff docs first, then simplify the script and manifests to match the chosen retention model.
- [Azure lifecycle policy is age-based, not count-based] -> Choose a retention window such as 56-60 days that approximates eight weekly backups and document that trade-off.
- [The simplified script may still grow again over time] -> Keep the script narrowly focused on upload and revisit image ownership only if logic expands again.

## Migration Plan

1. Write handoff docs in `homelab` that define the simplified Mealie backup script contract and the desired Azure Blob lifecycle policy for `mealie/files/`.
2. Write a Terraform handoff that defines the desired Azure Blob lifecycle policy for `mealie/files/`.
3. Simplify the Mealie backup script so it only creates the archive and uploads it.
4. Remove prune-related logic and any dependencies that exist only to support it.
5. Simplify the CronJob/config manifests as needed so they reflect the smaller script contract.
6. Apply the Azure lifecycle policy from Terraform.
7. Reconcile Flux and manually trigger the CronJob to verify upload behavior while Azure owns retention.

Rollback strategy:
- Revert the backup script and related ConfigMap/CronJob changes in `homelab`.
- Reintroduce the prior prune-capable script only if the Azure lifecycle policy path is not ready.
- Leave the Azure workload identity wiring untouched, since it is not part of this refactor.

## Open Questions

- None at this time.
