# Replicated Admin Playbook

Field notes, gotchas, and runbooks accumulated from hands-on experience operating a real application on the Replicated platform.

This isn't docs — it's the stuff the docs don't tell you, learned the hard way.

## Contents

| Skill | Description |
|---|---|
| [Releases](./releases.md) | How releases actually work, the `.replicated` config, OCI publishing |
| [Channels](./channels.md) | Channel strategy, promotion gates, CI enforcement |
| [Customers & Licenses](./customers-licenses.md) | License types, fields, entitlements, the SDK |
| [CMX Clusters](./cmx-clusters.md) | Provisioning, kubeconfig, ports, smoke testing |
| [Helm Chart Packaging](./helm-packaging.md) | Subcharts, values passdown, upgrade gotchas |
| [KOTS Config Screen](./kots-config.md) | Config CR, HelmChart mapping, ConfigOption |
| [RBAC & Service Accounts](./rbac-service-accounts.md) | Policies, service accounts, CI token scoping |
| [SDK Integration](./sdk-integration.md) | Correct API usage, field mapping, expiry, instance tracking |
| [Enterprise Portal](./enterprise-portal.md) | Branding, email templates, template variables |
| [Instance Tracking](./instance-tracking.md) | How instances are counted, SDK check-ins, stale instances |
| [Embedded Cluster (v2)](./embedded-cluster.md) | EC v2 config, storage classes, NodePort collisions |
| [Embedded Cluster v3](./embedded-cluster-v3.md) | EC v3 install flow, CLI changes, proxy domain gotcha, CMX VM testing |
| [Enterprise Portal v2](./enterprise-portal-v2.md) | Docs repo structure, toc.yaml, Terraform modules, Helm reference, versioning |

## Philosophy

- **Never install direct to cluster without going through Replicated.** Use `cluster prepare` or `helm install` from `registry.replicated.com`.
- **Helm upgrades are the delivery mechanism, not a workaround.** If you're hitting limits, fix the chart, not the process.
- **The SDK is the integration surface.** Read the actual docs, match the actual response shapes.
- **CI is the only path to Stable.** No human (or agent) promotes to Stable directly.
