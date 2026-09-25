# E1–E4 Lab Validation Plan

Context: Rancher 2.14.1 + Harvester 1.7.1 on rancher-single/harvester1 (work-synced).

## E1 — Cluster template drives the current node driver (the deciding experiment)

Fork/adapt `rancher/cluster-template-examples` chart; port the Harvester provider
values from the tofu reference (rs-lh1) to:

- a 1-node smoke cluster via **template UI flow** (Apps → create cluster → template)
- against **harvester1** using the current driver shapes

Open questions to answer:
1. Does the example chart's v1beta1-era provider shape
   (`cloudprovider: harvester`, `cloudCredentialSecretName`, `networkName`,
   `imageName`, `vmNamespace`, `sshUser`) still work, or must it be rewritten to the
   current `rke-machine.cattle.io` machine-config shape (which the tofu module encodes
   and we know works)?
2. Where do the pieces that the tofu tfvars carry map to?
   - `harvester_image_name` (VMI, k8s-infra ns) → template `imageName`?
   - `harvester_storage_class` (harvester-longhorn-single-mig, migratable=true) —
     VM-level property; is it exposed in current templates?
   - `harvester_vm_network` (homelan) → `networkName`
   - `ssh_user` (sles) + key/user_data delivery — does the template flow pass user_data?
   - cloud credential secret (cc-9vrmc / harvester1-171) → `cloudCredentialSecretName`
   - node pools (3 CP tainted + 3 workers, 6c/8G/80G vs 160G) → `nodepools[]`
   - single-disk pattern (longhorn_disk_gb = 0)
3. Does the rendered cluster come out managed like any other Rancher-launched cluster
   (same Machine/MachineDeployment CRs under the hood)?

## E2 — Template-upgrade semantics

Bump image in the chart's values → **Installed Apps** upgrade of an existing template-
created cluster. Does it roll nodes the way tofu rounds 1–2 did (machine replacement,
controller-gated drain)? Or is it a no-op / pitfall? (Docs: "no changes to the template
will affect the cluster" after provisioning — the App-upgrade path is the only
documented update; its node semantics are unproven.)

## E3 — Fleet-managed cluster from template (the 2.14-doc flow)

Template in `fleet-local` + `fleet.yaml` → "Create Cluster from Template" → confirm
Fleet takes over cluster management. Then have Fleet own Longhorn inside it (bundle in
`fleet/bundles/longhorn/`), with cluster *labels* doing per-customer targeting.

## E4 — Same-shape bake-off

Identical 2-pool cluster: tofu vs template-driven, timed and destructively tested
(kill a worker mid-churn, verify data continuity with the lh-test loop). Output feeds
the eventual "how we do it at work" doc with apples-to-apples numbers.

---
Baseline numbers from tofu (2026-09-25, rs-lh1): 6-node build Ready in ~9 min;
OS-image hop 6.0→6.1→6.2: 6 machine replacements, ~25–35 min per hop, zero data loss
(created-at `2026-09-25T14:28:54Z` survived 12 replacements; counters continuous;
replicas 3/3 through every churn).