# Cluster Templates

Rancher Cluster Templates are **Helm charts** installed into the `local` management
cluster (annotation `catalog.cattle.io/type: cluster-template`). A template carries
Kubernetes config + node-pool config in one chart; installing it creates a cluster
resource Rancher provisions. Versioning lives in this git repo (helm release history).

Per SUSE/Rancher docs (v2.14 > Cluster Management > Cluster Templates):

- Template changes do **not** affect already-provisioned clusters (one-shot
  instantiation); the update path is the **Installed Apps** upgrade of the template
  release.
- The "Deploying Clusters from a Template with Fleet" flow adds a `fleet.yaml` in this
  repo (template installed in `fleet-local`); clusters created from the template are
  then managed by Fleet.
- Rancher's example (chart-era 2021–2025) ships Harvester as a first-class provider
  (`values-harvester.yaml`: `cloudprovider: harvester`, `cloudCredentialSecretName`,
  `nodepools[]` with `etcd/controlplane/worker` roles, `quantity`, `diskSize`,
  `cpuCount`, `memorySize`, `networkName`, `imageName`, `vmNamespace`, `sshUser`).

## Status

- [ ] E1: template chart drives the current Harvester node driver (see
      `../docs/test-plan.md`)
- [ ] E2: template-upgrade node semantics
- [ ] E3: Fleet-managed cluster from template

## Values port (tofu → template)

`values-cluster-rs-lh1.yaml` carries the working rs-lh1 tofu tfvars, mapped onto the
template chart's value surface. Differences between the tofu harness and the template
value shape are the E1 questions (storage class, user_data, current driver fields).

## Chart

`chart/` is where the template Helm chart is built (start from
`rancher/cluster-template-examples`, adapt to the current driver + this repo's values).