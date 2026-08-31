# Deploying the Advanced Compute Platform

This guide walks through setting up AutoShift with the **Advanced Compute Platform** standards, splitting hub infrastructure and managed cluster policies into distinct ACM Governance groupings, and configuring HA and non-HA managed cluster profiles that share a common baseline.

## Architecture Overview

```
                        ACM Governance UI
                    ┌───────────────────────┐
                    │      Standards         │
                    ├───────────┬───────────┤
                    │   Hub     │  Managed  │
                    │ Standard  │ Standard  │
                    └─────┬─────┴─────┬─────┘
                          │           │
          ┌───────────────┘           └───────────────┐
          v                                           v
 ┌─────────────────────┐                 ┌─────────────────────────┐
 │ Advanced Compute    │                 │ Advanced Compute        │
 │ Platform Hub        │                 │ Platform                │
 ├─────────────────────┤                 ├─────────────────────────┤
 │ ACM                 │                 │ Virtualization          │
 │ ACS                 │                 │ Cert-manager            │
 │ GitOps / ArgoCD     │                 │ Logging / Loki          │
 │ Cluster install     │                 │ NMState                 │
 │ Cluster labels      │                 │ Compliance operator     │
 │ Cluster config maps │                 │ ...                     │
 │ OpenShift DNS       │                 ├────────────┬────────────┤
 │ MetalLB             │                 │  HA        │  Non-HA   │
 └─────────────────────┘                 │  Clusters  │  Clusters │
          │                              ├────────────┼────────────┤
          v                              │ ODF        │ LVM       │
     ┌─────────┐                         │ Storage    │ Local     │
     │   Hub   │                         │ nodes      │ storage   │
     │ Cluster │                         └──────┬─────┴─────┬─────┘
     └─────────┘                                v           v
                                          ┌──────────┐ ┌──────────┐
                                          │ prod-1   │ │ edge-1   │
                                          │ prod-2   │ │ edge-2   │
                                          │ ...      │ │ sno-1    │
                                          └──────────┘ └──────────┘
```

All managed clusters share the same **Advanced Compute Platform** standard in the ACM Governance UI. The HA vs. non-HA distinction is handled entirely by **placement labels** — different clusters get different storage policies, but they all appear under the same standard.

## Values File Layout

AutoShift composes configuration from multiple Helm values files. Each file defines a distinct top-level key, so they merge cleanly:

```
autoshift/values/
  global.yaml                 # Shared: git repo, branch, standards
  clustersets/
    hub.yaml                  # Hub clusterset definition
    ha-production.yaml        # HA managed clusters (ODF)
    non-ha-edge.yaml          # Non-HA managed clusters (LVM)
```

```
                     helm upgrade autoshift ./autoshift \
                       -f values/global.yaml             \
                       -f values/clustersets/hub.yaml     \
                       -f values/clustersets/ha-production.yaml \
                       -f values/clustersets/non-ha-edge.yaml

                              │
            Helm deep-merges  │  (each file owns a unique key)
                              v

     ┌──────────────────────────────────────────────────────┐
     │ autoshift:                                           │
     │   policyStandard: "Advanced Compute Platform"        │
     │   policyStandardHub: "Advanced Compute Platform Hub" │
     │                                                      │
     │ hubClusterSets:                                      │
     │   hub: { ... }                  <-- from hub.yaml    │
     │                                                      │
     │ managedClusterSets:                                  │
     │   ha-production: { ... }  <-- from ha-production.yaml│
     │   non-ha-edge: { ... }    <-- from non-ha-edge.yaml  │
     └──────────────────────────────────────────────────────┘
```

## Step 1: Global Configuration

Set both standards in `global.yaml`:

```yaml
# autoshift/values/global.yaml
autoshift:
  dryRun: false
  policyStandard: "Advanced Compute Platform"
  policyStandardHub: "Advanced Compute Platform Hub"

autoshiftGitRepo: https://github.com/your-org/autoshiftv2.git
autoshiftGitBranchTag: main
policyGenerator: true
```

## Step 2: Hub Clusterset

The hub runs ACM, ACS, GitOps, and other hub infrastructure. These policies appear under **Advanced Compute Platform Hub** in the Governance UI.

```yaml
# autoshift/values/clustersets/hub.yaml
hubClusterSets:
  hub:
    labels:
      self-managed: 'true'

      # Hub infrastructure
      acm: 'true'
      acm-channel: release-2.16
      acm-source: redhat-operators
      acm-source-namespace: openshift-marketplace
      acm-subscription-name: advanced-cluster-management

      acs: 'true'
      acs-channel: stable
      acs-source: redhat-operators
      acs-source-namespace: openshift-marketplace
      acs-subscription-name: rhacs-operator

      gitops: 'true'
      gitops-channel: gitops-1.21
      gitops-source: redhat-operators
      gitops-source-namespace: openshift-marketplace
      gitops-subscription-name: openshift-gitops-operator

      cert-manager: 'true'
      cert-manager-channel: stable-v1
      cert-manager-source: redhat-operators
      cert-manager-source-namespace: openshift-marketplace
      cert-manager-subscription-name: openshift-cert-manager-operator

      cluster-install: 'true'
      htpasswd: 'true'
      remove-kubeadmin: 'true'
```

## Step 3: HA Managed Clusters (ODF)

HA clusters get ODF for replicated block and object storage, plus storage nodes. All policies appear under **Advanced Compute Platform**.

```yaml
# autoshift/values/clustersets/ha-production.yaml
managedClusterSets:
  ha-production:
    labels:
      # ---- Storage: HA (ODF) ----
      odf: 'true'
      odf-channel: stable-4.22
      odf-source: redhat-operators
      odf-source-namespace: openshift-marketplace
      odf-subscription-name: odf-operator
      odf-resource-profile: balanced
      storage-nodes: '3'                  # triggers storage node MachineSet creation

      # ---- Common baseline ----
      virt: 'true'                        # OpenShift Virtualization
      virt-channel: stable
      virt-source: redhat-operators
      virt-source-namespace: openshift-marketplace
      virt-subscription-name: kubevirt-hyperconverged

      acs: 'true'                         # ACS SecuredCluster agent
      acs-channel: stable
      acs-source: redhat-operators
      acs-source-namespace: openshift-marketplace
      acs-subscription-name: rhacs-operator

      nmstate: 'true'
      nmstate-channel: stable
      nmstate-source: redhat-operators
      nmstate-source-namespace: openshift-marketplace
      nmstate-subscription-name: kubernetes-nmstate-operator

      logging: 'true'
      logging-channel: stable-6.5
      logging-source: redhat-operators
      logging-source-namespace: openshift-marketplace
      logging-subscription-name: cluster-logging

      cert-manager: 'true'
      cert-manager-channel: stable-v1
      cert-manager-source: redhat-operators
      cert-manager-source-namespace: openshift-marketplace
      cert-manager-subscription-name: openshift-cert-manager-operator
```

## Step 4: Non-HA Managed Clusters (LVM)

Non-HA clusters (compact, single-node, edge) use LVM for local storage. They share the common baseline with HA clusters but get a different storage stack.

```yaml
# autoshift/values/clustersets/non-ha-edge.yaml
managedClusterSets:
  non-ha-edge:
    labels:
      # ---- Storage: Non-HA (LVM) ----
      lvm: 'true'
      lvm-channel: stable-4.22
      lvm-source: redhat-operators
      lvm-source-namespace: openshift-marketplace
      lvm-subscription-name: lvms-operator

      # ---- Common baseline (same as HA) ----
      virt: 'true'
      virt-channel: stable
      virt-source: redhat-operators
      virt-source-namespace: openshift-marketplace
      virt-subscription-name: kubevirt-hyperconverged

      acs: 'true'
      acs-channel: stable
      acs-source: redhat-operators
      acs-source-namespace: openshift-marketplace
      acs-subscription-name: rhacs-operator

      nmstate: 'true'
      nmstate-channel: stable
      nmstate-source: redhat-operators
      nmstate-source-namespace: openshift-marketplace
      nmstate-subscription-name: kubernetes-nmstate-operator

      logging: 'true'
      logging-channel: stable-6.5
      logging-source: redhat-operators
      logging-source-namespace: openshift-marketplace
      logging-subscription-name: cluster-logging

      cert-manager: 'true'
      cert-manager-channel: stable-v1
      cert-manager-source: redhat-operators
      cert-manager-source-namespace: openshift-marketplace
      cert-manager-subscription-name: openshift-cert-manager-operator
```

## How Placement Separates HA from Non-HA

Both HA and non-HA clusters belong to the same **Advanced Compute Platform** standard. The difference is which storage policies target which clusters — controlled entirely by labels and placements:

```
                   Advanced Compute Platform (standard)
                   ─────────────────────────────────────

     Shared policies (target both HA + non-HA via their own labels):
     ┌──────────────────────────────────────────────────────┐
     │  virt: 'true'     -->  OpenShift Virtualization      │
     │  acs: 'true'      -->  ACS SecuredCluster            │
     │  nmstate: 'true'  -->  NMState                       │
     │  logging: 'true'  -->  Logging + Loki                │
     │  cert-manager     -->  Cert-manager                  │
     └──────────────────────────────────────────────────────┘

     Storage policies (mutually exclusive by label):
     ┌───────────────────────┐    ┌───────────────────────┐
     │  odf: 'true'          │    │  lvm: 'true'          │
     │  ┌─────────────────┐  │    │  ┌─────────────────┐  │
     │  │ ODF operator    │  │    │  │ LVM operator    │  │
     │  │ StorageCluster  │  │    │  │ LVMCluster      │  │
     │  │ Storage nodes   │  │    │  │ Default SC      │  │
     │  │ Default SC      │  │    │  └─────────────────┘  │
     │  │ Ceph CSI        │  │    │                       │
     │  └─────────────────┘  │    │  Targets: non-ha-edge │
     │                       │    └───────────────────────┘
     │  Targets: ha-production│
     └───────────────────────┘
```

A cluster in `ha-production` has `odf: 'true'` but no `lvm` label, so only the ODF policies match. A cluster in `non-ha-edge` has `lvm: 'true'` but no `odf` label, so only the LVM policies match. Both clusters see the same virtualization, ACS, logging, and other shared policies.

## Deploying

Install or upgrade with the composed values files:

```bash
helm upgrade --install autoshift ./autoshift \
  -n openshift-gitops \
  -f autoshift/values/global.yaml \
  -f autoshift/values/clustersets/hub.yaml \
  -f autoshift/values/clustersets/ha-production.yaml \
  -f autoshift/values/clustersets/non-ha-edge.yaml
```

After the Helm release updates:

1. The ApplicationSet picks up the new `POLICY_STANDARD` / `POLICY_STANDARD_HUB` env vars
2. ArgoCD re-renders all policy Applications through the CMP sidecar
3. The ACM Governance UI shows two standards:

```
  ┌─────────────────────────────────────────────────────────┐
  │                  ACM Governance                         │
  │                                                         │
  │  Filter by Standard:                                    │
  │  ┌──────────────────────────────────┐                   │
  │  │ [x] Advanced Compute Platform    │  77 policies      │
  │  │ [x] Advanced Compute Platform Hub│  42 policies      │
  │  └──────────────────────────────────┘                   │
  │                                                         │
  │  Filter by Category:                                    │
  │  ┌──────────────────────────────────┐                   │
  │  │ [ ] CM Configuration Management  │                   │
  │  │ [ ] CA Security Assessment...    │                   │
  │  │ [ ] IA Identification and Auth.. │                   │
  │  └──────────────────────────────────┘                   │
  └─────────────────────────────────────────────────────────┘
```

## Adding a New Managed Cluster Profile

To add a third profile (e.g., GPU-enabled clusters with a different operator set), create another values file with a new `managedClusterSets` key and add it to your Helm command:

```yaml
# autoshift/values/clustersets/gpu-compute.yaml
managedClusterSets:
  gpu-compute:
    labels:
      # Same baseline
      virt: 'true'
      virt-channel: stable
      virt-source: redhat-operators
      virt-source-namespace: openshift-marketplace
      virt-subscription-name: kubevirt-hyperconverged

      # Storage (HA for GPU workloads)
      odf: 'true'
      odf-channel: stable-4.22
      odf-source: redhat-operators
      odf-source-namespace: openshift-marketplace
      odf-subscription-name: odf-operator

      # GPU-specific
      node-feature-discovery: 'true'
      node-feature-discovery-channel: stable
      node-feature-discovery-source: redhat-operators
      node-feature-discovery-source-namespace: openshift-marketplace
      node-feature-discovery-subscription-name: nfd
```

No code changes needed — the existing policies and placements handle it automatically based on the labels.

## Label Reference (Common Policies)

| Label | Policy | Description |
|-------|--------|-------------|
| `odf: 'true'` | openshift-data-foundation | HA replicated storage (Ceph) |
| `lvm: 'true'` | lvm | Local volume manager for non-HA |
| `virt: 'true'` | openshift-virtualization | KubeVirt / VM workloads |
| `acs: 'true'` | advanced-cluster-security | Stackrox security |
| `cert-manager: 'true'` | cert-manager | Certificate management |
| `logging: 'true'` | logging | Cluster logging |
| `nmstate: 'true'` | nmstate | Network configuration |
| `compliance: 'true'` | openshift-compliance-operator | Compliance scanning |
| `pipelines: 'true'` | openshift-pipelines | Tekton pipelines |
| `storage-nodes: '<count>'` | storage-nodes | Dedicated storage MachineSets |
| `local-storage: 'true'` | local-storage | Local storage operator |

## Hub-of-Hubs Deployment

The two-standard model works with hub-of-hubs without any changes. Each AutoShift instance is independent — it has its own Helm values, its own `policyStandard`/`policyStandardHub`, and its own ACM scope. The "hub vs. managed" distinction is always relative to the instance deploying it.

### Tier Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│ Tier 0: Hub of Hubs                                                  │
│                                                                      │
│  AutoShift app: "autoshift"                                         │
│  policyStandardHub: "Advanced Compute Platform Hub"                 │
│    → ACM, GitOps, cluster-install on HoH itself                    │
│                                                                      │
│  policyStandard: "Advanced Compute Platform"                        │
│    → ODF, logging, virt, etc. targeting hub1, hub2                  │
│    (spoke hubs are "managed clusters" from HoH's perspective)       │
│                                                                      │
│  HoH ACM sees: local-cluster (HoH), hub1, hub2                     │
└───────────────────┬───────────────────────────┬──────────────────────┘
                    │                           │
  ┌─────────────────▼──────────────┐  ┌─────────▼────────────────────┐
  │ Tier 1: Spoke Hub (hub1)       │  │ Tier 1: Spoke Hub (hub2)     │
  │                                │  │                              │
  │  AutoShift app: "hub1"         │  │  AutoShift app: "hub2"       │
  │  policyStandardHub: "..."      │  │  policyStandardHub: "..."    │
  │    → dormant (hub1 has no      │  │    → dormant                 │
  │      local-cluster in its ACM) │  │                              │
  │                                │  │  policyStandard: "..."       │
  │  policyStandard: "..."         │  │    → policies for spoke3,    │
  │    → policies for spoke1,      │  │      spoke4                  │
  │      spoke2                    │  └──────────────────────────────┘
  │                                │
  │  hub1 ACM sees: spoke1, spoke2 │
  └──────────┬──────────┬──────────┘
             │          │
        ┌────▼───┐ ┌────▼───┐
        │spoke1  │ │spoke2  │   Tier 2: Managed clusters
        └────────┘ └────────┘
```

### Why No Third Standard Is Needed

A spoke hub (hub1) plays two roles, but they are served by two different AutoShift instances that never overlap:

| Role of hub1 | Managed by | AutoShift instance | Standard used |
|---|---|---|---|
| Receiving operators (as a managed cluster) | HoH's ACM | HoH AutoShift | HoH's `policyStandard` |
| Deploying policies to its own spokes | hub1's own ACM | hub1 AutoShift | hub1's `policyStandard` |

No single AutoShift instance ever needs to distinguish "spoke hub targets" from "leaf cluster targets" — the HoH instance targets spoke hubs, and hub1's instance targets leaf spokes. They never mix.

### Dormant Hub Policies on Spoke Hubs

Hub1's AutoShift generates Applications for all policy directories (including ACM, GitOps, etc.) because the ApplicationSet discovers them from git. But the hub-targeting policies use Placements that look for clusters with labels like `acm: 'true'` among hub1's managed clusters. Hub1's ACM only sees spoke1 and spoke2, not itself — so if the spokes don't carry those labels, the hub policies are harmless no-ops. They exist in ArgoCD but match no clusters in ACM.

### Per-Tier Configuration

Each tier sets its standards independently via its own values files:

```yaml
# HoH values
autoshift:
  policyStandard: "Advanced Compute Platform"
  policyStandardHub: "Advanced Compute Platform Hub"

# hub1 values — can match HoH or use regional names
autoshift:
  policyStandard: "Region East Platform"
  policyStandardHub: "Region East Platform Hub"

# hub2 values
autoshift:
  policyStandard: "Region West Platform"
  policyStandardHub: "Region West Platform Hub"
```

### Global Hub Visibility

If Multicluster Global Hub is deployed on the HoH, it syncs policy status and metadata (including standard annotations) from all tiers via Kafka. The HoH Global Hub dashboard shows policies from every tier grouped by their standard annotations — no extra configuration needed.

```
  Multicluster Global Hub Dashboard (HoH)
  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  Policies by Standard:                                  │
  │  ┌───────────────────────────────────────────────────┐  │
  │  │ Advanced Compute Platform Hub  │  12 policies     │  │
  │  │   (HoH self-management)        │                  │  │
  │  ├────────────────────────────────┼──────────────────┤  │
  │  │ Advanced Compute Platform      │  34 policies     │  │
  │  │   (HoH → hub1, hub2)          │                  │  │
  │  ├────────────────────────────────┼──────────────────┤  │
  │  │ Region East Platform           │  28 policies     │  │
  │  │   (hub1 → spoke1, spoke2)     │                  │  │
  │  ├────────────────────────────────┼──────────────────┤  │
  │  │ Region West Platform           │  28 policies     │  │
  │  │   (hub2 → spoke3, spoke4)     │                  │  │
  │  └───────────────────────────────┴──────────────────┘  │
  │                                                         │
  │  Standards propagate as Policy annotations — synced     │
  │  automatically via Kafka, no PolicySets needed.         │
  └─────────────────────────────────────────────────────────┘
```

This cross-tier visibility is an advantage of standards over PolicySets — PolicySets are per-hub objects with no cross-hub aggregation.
