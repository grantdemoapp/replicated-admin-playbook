# CMX Clusters (Compatibility Matrix)

## Provisioning a Cluster

```bash
CLUSTER_ID=$(replicated cluster create \
  --app my-app \
  --distribution k3s \
  --version 1.32.3 \
  --name my-cluster \
  --ttl 4h \
  --output json | jq -r '.id')

# Wait for running
replicated cluster ls --app my-app

# Get kubeconfig (use --output-path, not redirect — redirect captures stderr too)
replicated cluster kubeconfig $CLUSTER_ID --app my-app --output-path /tmp/kubeconfig.yaml
```

## Installing via Replicated Registry

```bash
# Login with license credentials (username=licenseID, password=full license YAML)
LICENSE_ID=$(grep licenseID /tmp/license.yaml | awk '{print $2}')
cat /tmp/license.yaml | helm registry login registry.replicated.com \
  --username "$LICENSE_ID" --password-stdin

# Install — no --version flag, pull channel's current version
helm install my-app oci://registry.replicated.com/my-app/unstable/my-chart \
  --kubeconfig /tmp/kubeconfig.yaml \
  --namespace default \
  --set "replicated.license=$(cat /tmp/license.yaml)" \
  --wait --timeout 5m
```

## Exposing Ports

```bash
replicated cluster port expose $CLUSTER_ID --port 30001 --protocol https --app my-app

# Get the hostname (not "exposedPort" — the JSON field is "hostname")
HOSTNAME=$(replicated cluster port ls $CLUSTER_ID --app my-app --output json | \
  jq -r '.[] | select(.exposed_ports[]?.protocol == "https") | .hostname')
```

## Cleanup

```bash
replicated cluster rm $CLUSTER_ID --app my-app
```

## Stale CI Clusters

If a CI smoke test fails before the cleanup step, the cluster hangs around. Clean up manually:

```bash
replicated cluster ls --app my-app --output json | \
  jq -r '.[] | select(.name | startswith("ci-smoke")) | .id' | \
  xargs -I{} replicated cluster rm {} --app my-app
```

## Instance Pollution

Each CMX cluster provisioned with a real customer license creates a new instance record in the vendor portal. Use a dedicated test license for CI, not your real customer licenses.
