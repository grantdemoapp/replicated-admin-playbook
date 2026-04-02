# Embedded Cluster

## EmbeddedCluster Config Manifest

```yaml
apiVersion: embeddedcluster.replicated.com/v1beta1
kind: Config
spec:
  version: 2.14.1+k8s-1.34   # EC binary + k8s version
  roles:
    controller:
      name: app
      labels:
        app: "true"
  domains:
    proxyRegistryDomain: images.myapp.io    # custom proxy domain
    replicatedAppDomain: updates.myapp.io   # custom update endpoint domain
```

## Storage Class

EC ships with OpenEBS. The default StorageClass is `openebs-hostpath`.

**Do not hardcode this in the Helm chart** — it won't exist on standard k8s clusters. Set it only in the KOTS Config CR default:

```yaml
# manifests/config.yaml
- name: postgres_storage_class
  title: Storage Class
  type: text
  default: "openebs-hostpath"   # correct for EC
  help_text: "Leave blank to use cluster default on standard k8s"
```

```yaml
# demoapp-chart/values.yaml
postgres:
  storage:
    storageClass: ""   # blank = cluster default for helm installs
```

## NodePort Collisions

EC allocates some NodePorts for its own components. Port `30000` is used. Common safe ports for app services:
- `30001` — main app
- `30002` — secondary services

Avoid `30000`. Consider using `ClusterIP` + kubectl port-forward for internal tools (like pgweb) rather than NodePort.

## Checking the EC Installer

```bash
replicated channel inspect Unstable --app my-app | grep hasECInstaller
```

## EC Version in the Vendor Portal

The "Update available: v0.1.4" banner in the EC Admin Console may show an EC binary version, not your app version. This is separate from the Replicated SDK's `/app/updates` endpoint. Don't confuse the two:

- SDK `/app/updates` → your app's available releases
- EC Admin Console update banner → may refer to EC infrastructure version