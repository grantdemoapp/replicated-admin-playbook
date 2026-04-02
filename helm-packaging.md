# Helm Chart Packaging

## Subcharts

Declare local subcharts in `Chart.yaml` with `repository: ""` — Helm resolves them from `charts/`:

```yaml
dependencies:
  - name: postgres
    repository: ""
    version: "1.0.0"
  - name: replicated
    repository: oci://registry.replicated.com/library
    version: 1.18.2
```

Run `helm dependency update` to resolve and update `Chart.lock`.

## Values Passdown to Subcharts

Parent values flow to subcharts via the subchart name as a top-level key:

```yaml
# parent values.yaml
postgres:
  storage:
    size: 10Gi   # reaches postgres subchart as .Values.storage.size
```

Subcharts cannot read each other's values. If pgweb needs postgres connection details, duplicate them under the pgweb key in the parent.

## StatefulSet VolumeClaimTemplate Immutability

Adding or changing annotations on a StatefulSet's `volumeClaimTemplates` (e.g. `helm.sh/resource-policy: keep`) counts as a spec change. Kubernetes rejects it:

```
cannot patch "my-postgres": StatefulSet.apps is invalid: spec: Forbidden
```

Fix: orphan-delete the StatefulSet first. The pod and PVC survive.

```bash
kubectl delete statefulset my-postgres -n default --cascade=orphan
helm upgrade ...
```

## Service Type Changes (NodePort → ClusterIP)

Kubernetes won't let you change a Service's type in-place. Delete it first:

```bash
kubectl delete svc my-service -n default
helm upgrade ...
```

## The 1MB Helm State Limit

Passing a large license YAML via `--set replicated.license=...` can push the Helm release Secret over Kubernetes' 1MB limit during upgrades. Symptoms: upgrade succeeds but subsequent upgrades fail with "etcd object size limit exceeded".

Fix: orphan-delete the relevant resources and reinstall rather than upgrading in place, or use a `--set-file` instead of inline `--set`.

## Resource Orphaning on Subchart Migration

When moving a resource from the parent chart's `templates/` to a subchart, Helm loses ownership of it. The resource remains on the cluster but Helm no longer manages it. On the next upgrade, the subchart tries to create the same resource and hits a conflict.

Fix: manually delete the orphaned resource before upgrading.

## `helm.sh/resource-policy: keep` on VolumeClaimTemplates

Add this annotation to preserve PVCs across `helm delete`:

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
      annotations:
        helm.sh/resource-policy: keep
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: {{ .Values.storage.size }}
```

## openebs-hostpath for Embedded Cluster

EC ships with `openebs-hostpath` as the default StorageClass. Set this as the KOTS Config default — **not** as the Helm chart default:

```yaml
# manifests/config.yaml — default for EC installs
- name: postgres_storage_class
  default: "openebs-hostpath"
```

```yaml
# demoapp-chart/values.yaml — empty for standard k8s installs
storage:
  storageClass: ""
```
