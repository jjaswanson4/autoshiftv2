# PolicySets vs. Standards Annotations

AutoShift uses **standards annotations** to group policies in the ACM Governance UI. This doc explains why, compares the two mechanisms, and considers hub-of-hubs implications.

## At a Glance

| | Standards Annotations | PolicySets |
|---|---|---|
| **What it is** | Free-form string on Policy metadata | Kubernetes CRD (`PolicySet`) |
| **ACM Governance UI** | Filterable under "Standards" | Filterable under "Policy sets" |
| **Placement** | None — each Policy keeps its own | Has its own Placement + PlacementBinding |
| **Compliance status** | Per-policy only | Aggregated across all member policies |
| **Lifecycle** | Annotation — no object to manage | Kubernetes object with status, events, etc. |
| **Cross-hub (hub-of-hubs)** | Propagates as annotation on Policy | Per-hub only — no cross-hub aggregation |
| **AutoShift integration** | `${POLICY_STANDARD}` / `${POLICY_STANDARD_HUB}` substitution | Would require a new resource outside the per-policy pipeline |

## Detailed Comparison

### Grouping and UI

Both mechanisms create filterable groups in the ACM Governance dashboard, but in different sections:

```
  ACM Governance UI
  ┌────────────────────────────────────────────┐
  │                                            │
  │  Filter by Standard:          <── annotations
  │  ┌──────────────────────────┐              │
  │  │ Advanced Compute Platform│              │
  │  │ Advanced Compute P. Hub  │              │
  │  │ NIST SP 800-53           │              │
  │  └──────────────────────────┘              │
  │                                            │
  │  Filter by Policy set:       <── PolicySets
  │  ┌──────────────────────────┐              │
  │  │ (none defined)           │              │
  │  └──────────────────────────┘              │
  │                                            │
  │  Filter by Category:         <── annotations
  │  ┌──────────────────────────┐              │
  │  │ CM Configuration Mgmt    │              │
  │  │ CA Security Assessment   │              │
  │  │ IA Identification & Auth │              │
  │  └──────────────────────────┘              │
  └────────────────────────────────────────────┘
```

Standards give you a three-level taxonomy (standard / category / control) in a single annotation set. PolicySets give you a flat grouping with aggregated compliance status.

### Placement

This is the key architectural difference:

```
  Standards (current)                    PolicySets
  ───────────────────                    ──────────

  Policy A ──┐                           PolicySet ──── Placement (group)
    own Placement (odf: true)            │
  Policy B ──┤                           ├── Policy A
    own Placement (lvm: true)            │     own Placement (odf: true)  ← conflict
  Policy C ──┘                           ├── Policy B
    own Placement (virt: true)           │     own Placement (lvm: true)  ← conflict
                                         └── Policy C
  Each policy independently                   own Placement (virt: true)  ← conflict
  targets clusters by label.
                                         PolicySet wants to own placement
                                         for the group, but each policy
                                         already has its own targeting.
```

AutoShift's design is that **each policy owns its placement** via label selectors (`autoshift.io/odf`, `autoshift.io/lvm`, etc.). PolicySets expect to provide group-level placement, which conflicts with per-policy targeting.

When a policy belongs to a PolicySet, PolicyGenerator **stops generating individual placement** for that policy unless `generatePlacementWhenInSet: true` is set — creating a default-off behavior that would need to be overridden on every policy.

### Compliance Status

| Capability | Standards | PolicySets |
|---|---|---|
| Per-policy compliance | Yes (native) | Yes (native) |
| Group compliance (single field) | No | Yes (`status.compliant`) |
| Queryable via `oc get` | No (annotation only) | Yes (CRD with status) |
| Automation trigger | Must query individual policies | Can watch one object |

PolicySets' aggregated status is useful for automation ("are all platform policies compliant?"), but AutoShift doesn't currently use that pattern.

## Hub-of-Hubs Considerations

In a hub-of-hubs architecture, AutoShift bootstraps each tier independently:

```
  ┌─────────────────────────────────────────────────────┐
  │                   Hub of Hubs                        │
  │                                                      │
  │  AutoShift bootstrap (tier 0)                        │
  │  Policies: ACM, GitOps, cluster-install              │
  │  Standard: "Advanced Compute Platform Hub"           │
  │                                                      │
  │  PolicySet?  Only sees policies in THIS namespace.   │
  │              Cannot aggregate spoke hub policies.    │
  └──────────┬────────────────────────┬──────────────────┘
             │                        │
             v                        v
  ┌──────────────────┐     ┌──────────────────┐
  │    Spoke Hub 1   │     │    Spoke Hub 2   │
  │                  │     │                  │
  │  AutoShift       │     │  AutoShift       │
  │  bootstrap       │     │  bootstrap       │
  │  (tier 1)        │     │  (tier 1)        │
  │                  │     │                  │
  │  Own policies,   │     │  Own policies,   │
  │  own namespace   │     │  own namespace   │
  └────────┬─────────┘     └────────┬─────────┘
           │                        │
           v                        v
      ┌─────────┐             ┌─────────┐
      │ Managed │             │ Managed │
      │clusters │             │clusters │
      └─────────┘             └─────────┘
```

| Behavior | Standards | PolicySets |
|---|---|---|
| Propagation across tiers | Annotations travel with the Policy object | PolicySet is a separate object, stays on the hub that created it |
| Visibility at top hub | Multicluster Global Hub syncs policy status + annotations via Kafka — standards are visible | PolicySet aggregation is per-hub only; no cross-hub PolicySet status |
| Spoke hub independence | Each spoke hub's `autoshift.policyStandard` can differ | Each spoke hub would need its own PolicySet objects |
| Unified dashboard | Global Hub dashboard shows policies grouped by standard across all tiers | No cross-tier PolicySet view exists |

**Standards annotations propagate naturally** because they're part of the Policy object. The Multicluster Global Hub agent syncs policy compliance status and metadata (including annotations) from spoke hubs to the global hub, making standards-based filtering work across tiers without extra configuration.

**PolicySets don't propagate cross-hub.** A PolicySet on Spoke Hub 1 is invisible to the Hub of Hubs. You'd need to create and maintain duplicate PolicySets at each tier, with no way to aggregate them into a single compliance view at the top.

## When PolicySets Would Make Sense

PolicySets are the right tool when:

| Use Case | Why PolicySets | AutoShift Equivalent |
|---|---|---|
| Deploy a bundle of policies as a unit to specific clusters | PolicySet owns placement for the group | Per-policy placement via labels (already works) |
| Gate an action on group compliance ("all PCI policies must be compliant before upgrade") | Watch `PolicySet.status.compliant` | Would need to query individual policies |
| Admin-curated policy bundles outside GitOps | PolicySet is a standalone object | Not the AutoShift model |

If AutoShift later needs programmatic group compliance checks (e.g., a Job that waits for all platform policies to be compliant before proceeding), PolicySets could be added as a **thin overlay** — referencing existing policies by name without changing their placement. This would complement standards, not replace them.

## Recommendation

**Use standards annotations** (current approach) for these reasons:

1. **No placement conflict** — each policy keeps its own label-driven targeting
2. **Hub-of-hubs ready** — annotations propagate with policies and are visible in the Global Hub dashboard
3. **Zero infrastructure** — no additional CRDs to manage, no lifecycle to maintain
4. **Configurable at deploy time** — `autoshift.policyStandard` / `autoshift.policyStandardHub` via existing `${...}` substitution
5. **Three-level taxonomy** — standard + category + control gives richer filtering than PolicySets' flat grouping

PolicySets remain available as a future addition for compliance automation if needed, without requiring changes to the current standards-based grouping.
