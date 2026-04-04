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

## `readonly: true` — Generated Passwords That Must Survive Upgrades

**The failure mode:** `RandomString` in a `default` field regenerates every time KOTS evaluates the config — including on upgrade. If a database was initialized with the first generated password, the second upgrade will write a different value to the Helm secret, breaking the DB connection permanently (the PVC data still expects the original password).

**The fix:** add `readonly: true` to any config item that uses `RandomString` as its default:

```yaml
- name: postgres_password
  title: Database Password
  type: password
  default: '{{repl RandomString 32}}'
  readonly: true   # ← critical: generated once on first install, never changed again
  hidden: true
```

**When `readonly: true` applies:**
- Any config item whose value initializes persistent state (database passwords, encryption keys, JWT secrets)
- Anything where changing the value post-install would break or corrupt the running app
- `RandomString` defaults are the most common case, but the same applies to any auto-generated default that must be stable

**When it does NOT apply:**
- User-configurable fields where changing the value is intentional (storage size, hostnames, feature flags)
- Fields that don't affect persistent state

**Subchart passthrough:** if a subchart (e.g. pgweb) needs the same generated password as the main DB, wire it through the HelmChart CR explicitly:

```yaml
# manifests/demoapp-helmchart.yaml
values:
  pgweb:
    postgres:
      password: '{{repl ConfigOption "postgres_password" }}'
  postgresql:
    auth:
      password: '{{repl ConfigOption "postgres_password" }}'
```

Without this, the subchart uses its own hardcoded default and fails to authenticate against a DB that was initialized with the KOTS-generated password.

## License Type Restriction Gap

RBAC has no sub-permission for license type. `kots/app/*/license/create` is all-or-nothing — you cannot restrict "only create dev licenses" at the RBAC level. This is a known gap.
## Boolean Config Values Are Strings in Helm Templates

KOTS passes `bool` config values to Helm as the strings `"true"` or `"false"`, not Go booleans. This means any template using `{{- if .Values.someFlag }}` will evaluate the string `"false"` as **truthy** and render the block.

**Symptom:** a resource you expected to be disabled (e.g. an Ingress with `enabled: false`) still renders and may fail validation with an empty host or missing TLS config.

**Fix:** guard with `ne (toString ...)` instead of a bare `if`:

```yaml
{{- if and .Values.ingress.enabled (ne (toString .Values.ingress.enabled) "false") }}
```

**Apply this pattern to any Helm template that conditionally renders a resource based on a KOTS bool config item.** The same applies to nested booleans like `ingress.tls.enabled`.

Note: this is the same reason `postgres.embedded` uses `eq (toString .Values.postgres.embedded) "true"` throughout the chart — it was already correct there. New boolean-toggled resources need the same treatment.
