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
## E1 RESULT — PASS (2026-09-25, rancher-single 2.14.1 + harvester1 1.7.1)

Charted in cluster-templates/chart/ (rke2-harvester 0.1.0). Renders the CURRENT
provisioning.cattle.io/v1 Cluster + rke-machine-config.cattle.io/v1 HarvesterConfig
shapes (ported from live rs-lh1). Cluster rs-ct1 (1 all-role node) provisioned:

- helm install -> Cluster CR (t0); VMI Running on harvester1 at t+45s (IP .152,
  k8s-infra/homelan, 6.2-v3g image); machine Provisioned t+180s; cluster
  Provisioned/Connected/AgentDeployed/Ready; node Ready + cattle-cluster-agent +
  CCM (harvester-cloud-provider) + calico/traefik + system-upgrade-controller
  running at ~10-12 min.
- Chart values --> cloud-init chain verified end-to-end (ssh as sles with cwsneezy
  key works; sudo present).

Answers from E1:
- dynamicSchemaSpec NOT required in template machinePools (omitted -> worked).
- Storage class is NOT template-driven: VMs land on the Harvester default SC
  (harvester-longhorn-single-mig on h1). Parity with work clickops: fine.
- cloudCredentialSecretName references the Rancher credential (cattle-global-data:cc-9vrmc).
- Helm upgrade with changed chart values rolled the machine automatically
  (Deleting/Provisioning pair) — E2 signal: config delta -> machine replacement.

Pitfalls:
- Local-cluster kubeconfig secrets embed the internal service IP (10.43.x.x,
  unroutable off-cluster): rewrite server to the public URL +
  --insecure-skip-tls-verify for lab probes.
- Helm template rendering: YAML parses numbers as float64 -> use %.0f in printf,
  not %d (produces %!d(float64=N) corruption).
- Stale example chart (rancher/cluster-template-examples) predates the current
  driver field set; port the values from live objects, dont copy its values shape.
## E1 RESULT — PASS (2026-09-25, rancher-single 2.14.1 + harvester1 1.7.1)

Charted in cluster-templates/chart/ (rke2-harvester 0.1.0). Renders the CURRENT
provisioning.cattle.io/v1 Cluster + rke-machine-config.cattle.io/v1 HarvesterConfig
shapes (ported from live rs-lh1). Cluster rs-ct1 (1 all-role node) provisioned:

- helm install -> Cluster CR (t0); VMI Running on harvester1 at t+45s (IP .152,
  k8s-infra/homelan, 6.2-v3g image); machine Provisioned t+180s; cluster
  Provisioned/Connected/AgentDeployed/Ready; node Ready + cattle-cluster-agent +
  CCM (harvester-cloud-provider) + calico/traefik + system-upgrade-controller
  running at ~10-12 min.
- Chart values --> cloud-init chain verified end-to-end (ssh as sles with cwsneezy
  key works; sudo present).

Answers from E1:
- dynamicSchemaSpec NOT required in template machinePools (omitted -> worked).
- Storage class is NOT template-driven: VMs land on the Harvester default SC
  (harvester-longhorn-single-mig on h1). Parity with work clickops: fine.
- cloudCredentialSecretName references the Rancher credential (cattle-global-data:cc-9vrmc).
- Helm upgrade with changed chart values rolled the machine automatically
  (Deleting/Provisioning pair) — E2 signal: config delta -> machine replacement.

Pitfalls:
- Local-cluster kubeconfig secrets embed the internal service IP (10.43.x.x,
  unroutable off-cluster): rewrite server to the public URL +
  --insecure-skip-tls-verify for lab probes.
- Helm template rendering: YAML parses numbers as float64 -> use %.0f in printf,
  not %d (produces %!d(float64=N) corruption).
- Stale example chart (rancher/cluster-template-examples) predates the current
  driver field set; port the values from live objects, don't copy its values shape.