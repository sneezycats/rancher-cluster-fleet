# Fleet layer

Fleet = CRD GitOps engine: a Fleet **manager** (rancher-single's local cluster) with
**agents** on downstream clusters. A `GitRepo` CR points at this repo; each folder =
a **bundle** (manifests / kustomize / Helm chart), rendered as a helm release per
cluster (`BundleDeployment`). Targeting is by **cluster labels** and overlays.

## What lives here

- `bundles/longhorn/` — Longhorn as a Fleet bundle. Originated in the lab's
  `cwillis/fleet-longhorn` (Forgejo, now folded here). Validated against Rancher
  2.15.1/hobbyfarm on 2026-09-20; must be re-validated against the work-synced
  rancher-single (2.14.1) estate next.
- `fleet.yaml` — repo manifest Fleet uses to find bundles.

## Known reconciler warts (from lab validation, 2026-09-20)

1. `helm.targetNamespace`/`createNamespace` silently dropped in that Fleet version →
   releases land in the default namespace. Use top-level `defaultNamespace` as a plain
   STRING (object form is the newer Fleet).
2. `correctDrift` = object `{enabled: true}`.
3. Be explicit with paths, or deploy only from the repo root.
4. BundleDeployments live in `cluster-fleet-default-<cluster>-<hash>` namespaces.
5. After a failed helm adoption the agent stops reconciling even across restarts —
   delete the BundleDeployment in its cluster namespace; the bundle recreates it.
6. Longhorn's driver-deployer pod recreates deleted RBAC/storage classes — kill it
   BEFORE purging a botched install. (Longhorn-specific, keep in the bundle notes.)

Warts 1/5/6 hit us in the lab; they're reconciler-behavior realities, not blockers.

## Roadmap (see ../docs/test-plan.md)

- E3: template-created cluster handed to Fleet ("Create Cluster from Template" flow)
- Longhorn bundle re-validated on rancher-single (2.14.1)
- Later: observability bundles (grafana/node-exporter), per-customer labels