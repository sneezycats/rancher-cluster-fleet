# Fleet-managed Longhorn for Rancher-provisioned clusters

GitOps installation of Longhorn on RKE2 clusters via **Fleet** — the GitOps
controller already built into every Rancher 2.x management cluster. No extra
control plane, no agent registration step: every Rancher-provisioned cluster
automatically appears as a Fleet cluster target.

This repo is the candidate pattern for production use; it is deliberately
portable — see "Adapting to a new environment".

## Why Fleet (and not a second GitOps tool)

- **Already there.** Rancher ships and operates Fleet itself (`cattle-fleet-system`).
  Zero new infrastructure to run, back up, or upgrade.
- **Zero registration.** Provisioned clusters self-register with Fleet at
  creation. A cluster that is Ready in Rancher is a deployable Fleet target —
  no kubeconfig copying, no agent install (unlike external Argo CD, whose
  cluster registration has proven fragile).
- **Same release train.** Fleet is versioned and tested with Rancher; cluster
  upgrade tests exercise it for free.

## How it works

```
this git repo ──(GitRepo CR polls)──> Fleet controller (mgmt cluster)
     │                                      │
     │ fleet.yaml: chart pin + values       │ bundle → every cluster matching
     │ longhorn-values.yaml                 │ the clusterSelector
     ▼                                      ▼
 https://charts.longhorn.io ──────> longhorn-system on each target cluster
```

- `gitrepo.yaml` — the Fleet `GitRepo` resource. Applied **once** on the
  management cluster; it tells Fleet to watch this repo.
- `longhorn/fleet.yaml` — per-path Fleet config: pins the Longhorn chart
  **version** (deterministic upgrades = bump this one line) and points at the
  values file.
- `longhorn/longhorn-values.yaml` — the opinionated install values.

Fleet renders the helm chart itself (values-merge, no templating), creates
bundles per path, and applies them to all matching clusters. `correctDrift:
true` makes Fleet re-revert manual out-of-band changes — storage config stays
as committed.

## Prerequisites

1. **Rancher 2.x** with Fleet enabled (default) — verified healthy:
   `kubectl -n cattle-fleet-system get deploy` → `fleet-controller`, `gitjob` all 1/1.
2. **Target cluster(s) Ready in Rancher.** No further registration needed.
3. **iSCSI initiator on guest nodes** (Longhorn requirement). SL Micro ships
   it; verify on a node: `systemctl is-active iscsid`. If absent, bake
   `open-iscsi` into the golden image — do NOT try to deliver it via the same
   Fleet bundle (node-level packages precede k8s).
4. **Version governance**: pin the chart (below) to a version from the
   Longhorn↔Kubernetes support matrix for your K8s minor. Never track `latest`.

## Install (one-time per management cluster)

```bash
# 1. Label the target cluster(s) in Rancher (propagates to Fleet):
kubectl -n fleet-default patch clusters.fleet.cattle.io <CLUSTER-NAME> \
  --type=merge -p '{"metadata":{"labels":{"storage-longhorn":"true"}}}'

# 2. Point Fleet at this repo (edit the repo URL first):
kubectl apply -f gitrepo.yaml

# 3. Watch it land:
kubectl -n fleet-default get gitrepos,fleetclusters -w
kubectl -n longhorn-system --kubeconfig <guest-kubeconfig> get pods
```

## Upgrading Longhorn

1. Check the Longhorn↔K8s support matrix for every target cluster's K8s minor.
2. Bump `version:` in `longhorn/fleet.yaml`, commit, push.
3. Fleet polls (or push-syncs), re-renders, and rolls the chart out to all
   matching clusters. Per-cluster stagger: use separate `paths/` + labels
   (e.g. `canary` first) or split GitRepos.

## Uninstall / opt-out

Remove the cluster's `storage-longhorn` label (or edit `targets`) — Fleet stops
managing it. With `keepResources: false` (default) Fleet deletes what it
installed; set `keepResources: true` in `gitrepo.yaml` if you want Longhorn to
survive GitRepo removal (recommended for production!).

## Adapting to a new environment

| Placeholder | This lab | Where to change |
|---|---|---|
| Git repo URL | `https://git.home.sneezycats.com/sneezycats/fleet-longhorn.git` | `gitrepo.yaml` `spec.repo` |
| Cluster selector label | `storage-longhorn: "true"` | `gitrepo.yaml` `spec.targets` + cluster labels |
| Chart version | `1.12.1` | `longhorn/fleet.yaml` |
| Values | lab defaults, 3 replicas | `longhorn/longhorn-values.yaml` |

For work: same files; set `spec.repo` to the internal Forgejo/GitHub URL,
select clusters by your environment labels (`env: dev` / `clusterGroup: storage`),
and mirror any values differences per `paths/` directory (Fleet merges
per-path `fleet.yaml`, so `paths: [longhorn-dev]` vs `[longhorn-prod]` gives
per-environment values from one repo).

## Verification after rollout

```bash
kubectl -n longhorn-system get pods           # all Running
kubectl get storageclass                      # longhorn (default), longhorn-static
kubectl -n longhorn-system get settings.longhorn.io default-replica-count \
  -o jsonpath='{.value}'                      # 3
```
