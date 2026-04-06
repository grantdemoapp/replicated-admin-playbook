# Embedded Cluster v3

EC v3 is a complete redesign of Embedded Cluster. These notes are from hands-on experience building and deploying a real application on EC v3 (alpha).

## Config Manifest

```yaml
apiVersion: embeddedcluster.replicated.com/v1beta1
kind: Config
metadata:
  name: my-app
spec:
  version: "3.0.0-alpha-31+k8s-1.32"
  domains:
    proxyRegistryDomain: images.myapp.dev    # CRITICAL: set this if using a custom proxy domain
  roles:
    controller:
      name: app
      labels:
        app: "true"
```

**The version format changed**: EC v3 uses `3.0.0-alpha-XX+k8s-1.YY`, not the `2.x.x+k8s-1.YY` format from EC v2. The `+k8s-1.32` suffix selects the Kubernetes version.

## Key Differences from EC v2

- **CLI is the app slug**: The installer binary is named after your app (e.g. `./itrequest install`), not a generic `embedded-cluster` command.
- **`k0s kubectl` access**: Use `sudo <app-slug> shell -c "k0s kubectl get pods -A"` — there's no direct `kubectl` or `k0s` binary in PATH.
- **No interactive kubectl**: `shell` without `-c` requires a TTY. For automation, always use `shell -c "command"`.
- **Status command**: `sudo <app-slug> status` shows node readiness and version.
- **Support bundles**: `sudo <app-slug> support-bundle` generates a bundle.

## Install Flow (Online)

```bash
# 1. Download
curl -f "https://replicated.app/embedded/<app-slug>/<channel-slug>" \
  -H "Authorization: <license-id>" \
  -o <app-slug>-<channel>.tgz

# 2. Extract
tar -xvzf <app-slug>-<channel>.tgz

# 3a. Interactive install
sudo ./<app-slug> install --license license.yaml

# 3b. Headless install (CI/automation)
sudo ./<app-slug> install \
  --license license.yaml \
  --headless \
  --config-values config-values.yaml \
  --installer-password <password> \
  --ignore-host-preflights --yes
```

The `--ignore-host-preflights --yes` flags are useful for testing but should not be used in production.

## Headless Config Values

For headless installs, provide a ConfigValues manifest:

```yaml
apiVersion: kots.io/v1beta1
kind: ConfigValues
spec:
  values:
    postgres_embedded:
      value: "1"
    postgres_storage_size:
      value: "10Gi"
    app_service_type:
      value: "NodePort"
    app_node_port:
      value: "30001"
```

## Custom Proxy Domain — The `proxyRegistryDomain` Gotcha

**If you use a custom proxy domain** (e.g. `images.myapp.dev` instead of `proxy.replicated.com`), you **must** set `domains.proxyRegistryDomain` in the EC config. Without it:

1. The SDK creates `enterprise-pull-secret` with auth for `proxy.replicated.com` and `registry.replicated.com` only
2. Your images at `images.myapp.dev/proxy/...` will fail with `400 Bad Request` or auth errors
3. All pods except the SDK will be in `ImagePullBackOff`

Setting the domain tells EC to configure the pull secret for your custom domain.

## CMX VM Testing

EC v3 installs on bare VMs, not managed Kubernetes clusters. Use CMX VMs:

```bash
# Create a VM
replicated vm create --distribution ubuntu --version 24.04 \
  --name ec-test --ttl 4h --disk 50 --instance-type r1.medium

# SSH in (username depends on how keys are configured)
replicated vm ssh-endpoint <vm-id> --username <github-username>
ssh -p <port> <user>@<host>
```

**Important**: CMX VMs authenticate via GitHub SSH keys. The key must be on the GitHub account **before** the VM is created. Keys added after provisioning won't work — create a new VM.

**VM access shortcut**: `ssh <vm-id>@replicatedvm.com` uses the Replicated SSH gateway (requires GitHub username configured in vendor portal).

## KOTS Values as Strings

KOTS passes all config values to Helm as strings, including booleans and integers. This causes two problems:

### 1. Helm Schema Validation Failures

If your `values.schema.json` has `"type": "boolean"` or `"type": "integer"`, KOTS will fail with:
```
values don't meet the specifications of the schema(s):
- at '/ingress/enabled': got string, want boolean
- at '/service/nodePort': got string, want integer
```

**Fix**: Use union types in the schema:
```json
{
  "nodePort": { "type": ["integer", "string"] },
  "enabled": { "type": ["boolean", "string"] }
}
```

### 2. Template Conditionals

A bare `{{- if .Values.postgres.enabled }}` evaluates the string `"false"` as truthy.

**Fix**: Use string comparison:
```yaml
{{- if eq (toString .Values.postgres.enabled) "true" }}
```

Apply this to every conditional that's driven by a KOTS config value, including subcharts.

## What the Installer Outputs

After extraction, you'll see:
```
itrequest                          # CLI binary (named after app slug)
assets/itrequest-service           # Daemon binary
assets/itrequest-web               # Web UI binary
assets/release.tgz                 # The Helm chart + manifests
assets/ec-3.0.0-alpha-31+k8s-1.32.online  # EC infrastructure bundle
license.yaml                       # License file (if included in download)
```

## Built-in Extensions

EC v3 automatically installs:
- **k0s**: The Kubernetes distribution
- **OpenEBS**: `openebs-hostpath` StorageClass (use this as KOTS Config default)
- **Calico**: CNI networking
- **CoreDNS**: DNS
- **Metrics Server**: Resource metrics

## Limitations (Alpha)

- No disaster recovery support
- Some Replicated template functions not yet ported
- Worker profiles can only be set during initial install, not on upgrade
- Only one worker profile supported (applied to all nodes)
