# rancher-cluster-fleet

One Git home for the lab (== work-version-synced) Rancher estate's declarative surface:

- **`cluster-templates/`** — Helm-chart cluster templates that define whole Rancher
  clusters (Kubernetes config + node pools + Harvester provider wiring). Mirrors the
  Rancher Cluster Templates mechanism (docs: Rancher Manager v2.14 > Cluster Management >
  Cluster Templates; Rancher example: `rancher/cluster-template-examples`).
- **`fleet/`** — Fleet GitOps content for everything installed **inside** clusters:
  Longhorn, dashboards, monitoring. Contents originated from the lab's
  `cwillis/fleet-longhorn` (Forgejo) which is subsumed by this repo.

## Why one repo?

1. The documented "Deploying Clusters from a Template with Fleet" flow expects the
   template repo itself to carry a `fleet.yaml` — SUSE treats them as one surface.
2. One PR surface = one audit trail. A customer-team engineer reviewing a cluster
   change sees the cluster definition *and* what runs inside it in one diff.
3. Values are shared between the two (image names, networks, node sizes) — split
   repos guarantee drift.

## Layout

```
cluster-templates/
  README.md                     # how templates work; what E1 must validate
  values-cluster-*.yaml        # one file per cluster shape (values ported from tofu)
  chart/                        # the template Helm chart (built in E1)
fleet/
  README.md                     # Fleet approach + known reconciler warts
  fleet.yaml                    # repo manifest for Fleet
  bundles/
    longhorn/                   # Longhorn as a Fleet bundle (from fleet-longhorn)
docs/
  test-plan.md                  # E1–E4 lab validation plan
```

## Version-sync note

Lab = Rancher 2.14.1 + Harvester 1.7.1 (rancher-single/harvester1) — identical to the
work environment. Findings transfer without translation. Rancher doc pages can lag;
treat docs as directional and the lab as the source of truth (per `docs/test-plan.md`).