# Releases

## How Releases Actually Work

A Replicated release is a bundle of manifests pushed to the vendor platform. For Helm-only apps it contains:
- A packaged Helm chart TGZ
- A `HelmChart` manifest (`kots.io/v1beta2`) telling Replicated which chart artifact to serve
- Optional: other KOTS manifests (`Config`, `EmbeddedCluster`, etc.)

The release is then promoted to a channel, which makes it available at `registry.replicated.com/<app-slug>/<channel-slug>/<chart-name>`.

## The `.replicated` Config File

The `.replicated` file at the repo root tells the CLI how to build a release when you run `replicated release create --yaml-dir .`:

```yaml
appSlug: my-app
charts:
  - path: ./my-chart        # directory OR path to a TGZ
manifests:
  - ./manifests/*.yaml
```

**Critical gotcha:** If you point `charts.path` at a source directory, the Replicated backend sometimes fails to index the OCI artifact correctly and `helm show chart oci://registry.replicated.com/...` keeps returning the old version. Always package to a TGZ first.

## The Correct Release Workflow

```bash
# 1. Package the chart as a TGZ
helm dependency update my-chart/
helm package my-chart/ --version X.Y.Z --app-version A.B.C -d /tmp/staging/

# 2. Stage manifests alongside it
cp manifests/*.yaml /tmp/staging/

# 3. Write a .replicated pointing at the TGZ
cat > /tmp/staging/.replicated << EOF
appSlug: my-app
charts:
  - path: ./my-chart-X.Y.Z.tgz
EOF

# 4. Create and promote
replicated release create \
  --yaml-dir /tmp/staging/ \
  --version X.Y.Z \
  --promote Unstable \
  --app my-app
```

## OCI Registry Propagation

After `release create`, the OCI tag at `registry.replicated.com` does not update instantly. Poll before testing:

```bash
while true; do
  VER=$(helm show chart oci://registry.replicated.com/my-app/unstable/my-chart 2>/dev/null | grep '^version:' | awk '{print $2}')
  echo "$VER"
  [ "$VER" = "X.Y.Z" ] && break
  sleep 5
done
```

## HelmChart Manifest Versioning

The `chartVersion` in the `HelmChart` manifest **must exactly match** the `version` in `Chart.yaml`. If they diverge, the backend silently fails to publish the OCI tag.

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

## CI Automation Pattern

In GitHub Actions:

```yaml
- name: Create Replicated release
  env:
    REPLICATED_API_TOKEN: ${{ secrets.REPLICATED_API_TOKEN }}
  run: |
    VERSION=$(grep '^version:' my-chart/Chart.yaml | awk '{print $2}')
    helm dependency update my-chart/
    helm package my-chart/ --version "$VERSION" -d /tmp/staging/
    cp manifests/*.yaml /tmp/staging/
    cat > /tmp/staging/.replicated << EOF
    appSlug: my-app
    charts:
      - path: ./my-app-${VERSION}.tgz
    EOF
    replicated release create \
      --yaml-dir /tmp/staging/ \
      --version "$VERSION" \
      --promote Unstable \
      --app my-app
```

The CI token should use a service account scoped to the Developer RBAC policy (no direct Stable promotion).
