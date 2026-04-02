# KOTS Config Screen

## The Config Custom Resource

Defines the fields shown in the KOTS Admin Console at install/upgrade time:

```yaml
apiVersion: kots.io/v1beta1
kind: Config
metadata:
  name: my-app
spec:
  groups:
    - name: database
      title: Database
      description: Configure the embedded PostgreSQL database.
      items:
        - name: postgres_storage_size
          title: Storage Size
          type: text
          default: "10Gi"
          required: true
          validation:
            regex:
              pattern: ^\d+(Mi|Gi|Ti)$
              message: Must be a valid size like 10Gi or 512Mi.

        - name: postgres_storage_class
          title: Storage Class
          type: text
          default: "openebs-hostpath"
```

Item types: `text`, `textarea`, `password`, `bool`, `select_one`, `select_many`, `file`, `label`

## Mapping to Helm Values via HelmChart CR

```yaml
apiVersion: kots.io/v1beta2
kind: HelmChart
metadata:
  name: my-app
spec:
  chart:
    name: my-app
    chartVersion: X.Y.Z
  values:
    postgres:
      storage:
        size: '{{repl ConfigOption "postgres_storage_size" }}'
        storageClass: '{{repl ConfigOption "postgres_storage_class" }}'
```

`ConfigOption` uses the `name` field from the Config item, not the `title`.

## Helm-only Installs (No KOTS)

The Config CR is a KOTS-only resource. For `helm install` without KOTS, values come from the chart's `values.yaml` defaults or `--set` flags. The `HelmChart` CR values mapping is ignored.

This means:
- Helm chart defaults should be sane for standard k8s installs
- KOTS Config defaults can override for specific environments (e.g. EC)

## License Type Restriction Gap

RBAC has no sub-permission for license type. `kots/app/*/license/create` is all-or-nothing — you cannot restrict "only create dev licenses" at the RBAC level. This is a known gap.
