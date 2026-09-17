# Upstream PR Progress

Tracking PRs from fork (`jjaswanson4/autoshiftv2`) to upstream (`auto-shift/autoshiftv2`).

Last updated: 2026-09-17

## Summary

| Status | Count |
|--------|-------|
| Merged | 3 |
| Open (CI passing) | 5 |
| Open (CI failing) | 1 |
| **Total** | **9** |

## Merged

| PR | Title | Branch | Merged |
|----|-------|--------|--------|
| #231 | feat: config-driven ODF StorageCluster and local-storage | `feature/config-driven-odf-local-storage` | 2026-09-11 |
| #234 | feat: add remove-kubeadmin policy | `feature/remove-kubeadmin-policy` | 2026-09-14 |
| #235 | feat: add gitops console plugin policy | `feature/gitops-console-plugin-policy` | 2026-09-14 |

## Open (CI Passing)

| PR | Title | Branch | Review Status |
|----|-------|--------|---------------|
| #232 | feat: config-driven AAP instance spec via rendered-config | `feature/config-driven-aap` | Awaiting review |
| #236 | feat: config-driven ACME ClusterIssuer for cert-manager | `feature/cert-manager-acme-issuer` | Comments from bcarr-rh |
| #237 | feat: configurable policy standard via ${POLICY_STANDARD} | `feature/configurable-policy-standard` | Awaiting review |
| #238 | feat: config-driven PerformanceProfile override for workload-partitioning | `feature/config-driven-performance-profile` | Awaiting review |
| #240 | feat: add unschedulable-control-plane policy for compact clusters | `feature/unschedulable-control-plane` | Awaiting review |

## Open (CI Failing)

| PR | Title | Branch | Issue |
|----|-------|--------|-------|
| #233 | feat: add htpasswd identity provider policy | `feature/htpasswd-policy` | Label contract: missing `autoshift.io/htpasswd` in `_example.yaml` |

## Pending Decisions

- **PR #233 (htpasswd)**: Decision pending on whether to remove inline Secret creation (like ACME in #236) or keep it since bcrypt hashes are one-way. See memory `project_pr233-htpasswd-secrets.md`.

## CI Notes

- `secret-scan` job always fails and is not a merge blocker.
- All other CI jobs must pass before merge.
