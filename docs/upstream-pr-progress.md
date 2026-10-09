# Upstream PR Progress

Tracking PRs from fork (`jjaswanson4/autoshiftv2`) to upstream (`auto-shift/autoshiftv2`).

Last updated: 2026-10-09

## Summary

| Status | Count |
|--------|-------|
| Merged | 4 |
| Open (CI passing) | 4 |
| Open (CI failing) | 1 |
| **Total** | **9** |

## Merged

| PR | Title | Branch | Merged |
|----|-------|--------|--------|
| [#231](https://github.com/auto-shift/autoshiftv2/pull/231) | feat: config-driven ODF StorageCluster and local-storage | `feature/config-driven-odf-local-storage` | 2026-09-11 |
| [#234](https://github.com/auto-shift/autoshiftv2/pull/234) | feat: add remove-kubeadmin policy | `feature/remove-kubeadmin-policy` | 2026-09-14 |
| [#235](https://github.com/auto-shift/autoshiftv2/pull/235) | feat: add gitops console plugin policy | `feature/gitops-console-plugin-policy` | 2026-09-14 |
| [#232](https://github.com/auto-shift/autoshiftv2/pull/232) | feat: config-driven AAP instance spec via rendered-config | `feature/config-driven-aap` | 2026-10-01 |

## Open (CI Passing)

| PR | Title | Branch | Review Status |
|----|-------|--------|---------------|
| [#236](https://github.com/auto-shift/autoshiftv2/pull/236) | feat: config-driven ACME ClusterIssuer for cert-manager | `feature/cert-manager-acme-issuer` | Comments from bcarr-rh, responded |
| [#237](https://github.com/auto-shift/autoshiftv2/pull/237) | feat: configurable policy standard via `${POLICY_STANDARD}` | `feature/configurable-policy-standard` | Awaiting review |
| [#238](https://github.com/auto-shift/autoshiftv2/pull/238) | feat: config-driven PerformanceProfile override for workload-partitioning | `feature/config-driven-performance-profile` | Awaiting review |
| [#240](https://github.com/auto-shift/autoshiftv2/pull/240) | feat: add unschedulable-control-plane policy for compact clusters | `feature/unschedulable-control-plane` | Awaiting review |

## Open (CI Failing)

| PR | Title | Branch | Issue |
|----|-------|--------|-------|
| [#233](https://github.com/auto-shift/autoshiftv2/pull/233) | feat: add htpasswd identity provider policy | `feature/htpasswd-policy` | Label contract: `autoshift.io/htpasswd` consumed by the policy but not declared in `_example.yaml`. Fix not yet pushed — last CI run 2026-09-04. |

## Pending Decisions

- **[#233](https://github.com/auto-shift/autoshiftv2/pull/233) (htpasswd)**: Whether to remove inline Secret creation (as done for ACME in [#236](https://github.com/auto-shift/autoshiftv2/pull/236)) or keep it, since bcrypt hashes are one-way and inline creation enables central user management from the hub.

## CI Notes

- The `secret-scan` job always fails and is not a merge blocker.
- The label contract test (`TestPipeline_EndToEnd`) parses `_example*.yaml` as YAML, so commented-out labels are invisible to it. Every label a policy consumes must be declared uncommented.
- All other CI jobs must pass before merge.
