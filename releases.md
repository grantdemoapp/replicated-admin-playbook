# Releases

## How Releases Actually Work

A Replicated release is a bundle of manifests pushed to the vendor platform. For Helm-only apps it contains:
- A packaged Helm chart (source directory or pre-built TGZ)
- A `HelmChart` manifest (`kots.io/v1beta2`) telling Replicated which chart to serve
- Optional: other KOTS manifests (`Config`, `EmbeddedCluster`, etc.)

The release is promoted to a channel, which makes it available at `registry.replicated.com/<app-slug>/<channel-slug>/<chart-name>`.

## The `.replicated` Config File

Lives at the repo root (or any parent directory — the CLI walks up). The CLI looks for `.replicated` first, then `.replicated.yaml`. Supports hierarchical merging in monorepos: parent configs merge with child configs (child wins on scalars, arrays accumulate).

```yaml
appSlug: "my-app"       # or appId
charts:
  - path: ./chart       # directory with Chart.yaml — auto-packaged by CLI
    chartVersion: "1.0.0"   # optional override
    appVersion: "1.0.0"     # optional override
    chartName: my-chart     # optional, must pair with chartVersion
manifests:
  - ./manifests/*.yaml  # glob patterns, doublestar supported
preflights:             # optional
  - path: ./preflights/spec.yaml
promoteToChannelIds: [] # optional
promoteToChannelNames: []
releaseLabel: ""
repl-lint:              # optional linter config
  version: 1
  linters:
    helm:
      disabled: false
```

**Key point:** When `charts.path` points to a directory containing a `Chart.yaml`, the CLI automatically runs `helm dependency update` and `helm package` internally — you don't need to pre-package to a TGZ. The CLI creates a temp staging directory, packages the chart, copies manifests, and calls `makeReleaseFromDir` on the result.

## Creating a Release

With `.replicated` in place at the repo root, the simplest invocation:

```bash
replicated release create --version 1.0.0 --promote Unstable
```

No `--yaml-dir`, `--chart`, or `--app` flags needed — all resolved from `.replicated`.

## HelmChart Manifest Versioning

The `chartVersion` in the `HelmChart` manifest **must exactly match** the `version` in `Chart.yaml`. If they diverge, the backend fails to publish the OCI tag correctly.

```yaml
# manifests/helmchart.yaml
apiVersion: kots.io/v1beta2
kind: HelmChart
metadata:
  name: my-app
spec:
  chart:
    name: my-app
    chartVersion: X.Y.Z   # must match Chart.yaml version exactly
```

## OCI Registry Propagation

After `release create`, the OCI tag at `registry.replicated.com` may not update instantly. This is a server-side indexing delay, not a client-side packaging issue. Poll before testing:

```bash
while true; do
  VER=$(helm show chart oci://registry.replicated.com/my-app/unstable/my-chart 2>/dev/null | grep '^version:' | awk '{print $2}')
  echo "$VER"
  [ "$VER" = "X.Y.Z" ] && break
  sleep 5
done
```

## CI Automation Pattern

In GitHub Actions, using the `.replicated` config:

```yaml
- name: Install Replicated CLI
  run: |
    curl -s https://api.github.com/repos/replicatedhq/replicated/releases/latest \
      | grep "browser_download_url.*linux_amd64.tar.gz" \
      | cut -d '"' -f 4 \
      | xargs curl -sL | tar xz -C /usr/local/bin replicated

- name: Create and promote release
  env:
    REPLICATED_API_TOKEN: ${{ secrets.REPLICATED_API_TOKEN }}
  run: |
    VERSION=$(grep '^version:' my-chart/Chart.yaml | awk '{print $2}')
    replicated release create \
      --version "$VERSION" \
      --promote Unstable
```

The CLI reads `.replicated` at the repo root, packages the chart from the source directory, and handles manifests via glob. No `helm package` step needed in CI.

The CI token should use a service account scoped to the Developer RBAC policy (no direct Stable promotion).

## Promoting to Stable

```bash
# List releases to find the sequence you want
replicated release ls --app my-app

# Promote (requires a token with Stable promotion permission)
replicated release promote <sequence> Stable --app my-app --version X.Y.Z
```
