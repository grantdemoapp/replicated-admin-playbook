# Enterprise Portal v2 — Docs, Terraform, and Helm Reference

EP v2 uses a GitHub repo to source documentation content. The repo structure drives the portal's navigation, page content, and integrations with Helm chart references and Terraform modules.

## Repo Structure

```
├── toc.yaml                    # Navigation structure & visibility rules
├── pages/                      # Markdown content pages
│   ├── getting-started.md
│   ├── installation/
│   │   ├── requirements.md
│   │   ├── linux.md           # EC install guide
│   │   ├── helm.md            # Helm install guide
│   │   └── airgap.md
│   ├── operations/
│   │   ├── upgrading.md
│   │   ├── backup.md
│   │   └── troubleshooting.md
│   ├── reference/
│   │   └── api.md
│   └── support/
│       ├── faq.md
│       └── contact.md
```

## toc.yaml — The Navigation File

This is the most important file. It defines navigation, page ordering, visibility rules, and integrations.

```yaml
navigation:
  - title: Getting Started
    icon: rocket
    page: pages/getting-started.md

  - title: Installation
    icon: download
    items:
      - title: Linux Installation
        page: pages/installation/linux.md
        visible_when:
          entitlements:
            - isEmbeddedClusterDownloadEnabled
      - title: Helm Installation
        page: pages/installation/helm.md
        visible_when:
          entitlements:
            - isHelmInstallEnabled

  - title: Reference
    icon: file-text
    items:
      - title: Helm Reference
        helm_chart:
          name: my-chart          # Auto-generated from chart values
        visible_when:
          entitlements:
            - isHelmInstallEnabled

  - title: Infrastructure
    icon: cloud
    items:
      - title: AWS
        terraform_module: github.com/myorg/my-terraform//modules/aws?ref=main
        visible_when:
          entitlements:
            - isAWSEnabled

overrides:
  home: pages/getting-started.md

hide_defaults:
  - features/branding
  - features/webhooks
```

### Key `toc.yaml` Features

| Feature | Syntax | What It Does |
|---------|--------|-------------|
| `page` | `page: pages/path.md` | Links to a markdown page |
| `helm_chart` | `helm_chart: { name: chart-name }` | Auto-generates Helm values reference from the chart |
| `terraform_module` | `terraform_module: github.com/org/repo//path?ref=branch` | Renders Terraform module docs |
| `visible_when` | `visible_when: { entitlements: [fieldName] }` | Gates visibility by license entitlements |
| `icon` | `icon: rocket` | Navigation icon (Lucide icon names) |

### Helm Chart Reference (Rubric 6.5)

Adding `helm_chart: { name: my-chart }` to the toc automatically generates a reference page from your chart's `values.yaml`. To have at least one field NOT documented (per the rubric), simply omit the `## @param` annotation comment from that field in `values.yaml`.

### Terraform Modules (Rubric 6.6)

The `terraform_module` entry points to a Terraform module in a GitHub repo. The portal automatically renders the module's variables, outputs, and usage. The modules don't need to actually work — a well-structured "fake" module with proper `variables.tf` and `outputs.tf` is sufficient per the rubric.

## Template Variables

Pages use Handlebars-style templates:

| Variable | Description |
|----------|-------------|
| `{{ app.name }}` | Application name |
| `{{ app.slug }}` | Application slug |
| `{{ customer.name }}` | Customer name |
| `{{ customer.email }}` | Customer email |
| `{{ license.id }}` | License ID |
| `{{ channel.name }}` | Channel name |
| `{{ channel.slug }}` | Channel slug |
| `{{ release.version }}` | Current release version |
| `{{ entitlements.fieldName }}` | License entitlement value |

### Conditional Blocks

```handlebars
{{#if entitlements.isEmbeddedClusterDownloadEnabled}}
Content only visible to EC-enabled customers.
{{/if}}
```

## Versioning Strategy — Branches Per Release

**The docs repo must have a branch for each release version label.** The portal serves docs from the branch matching the customer's installed version.

### Pattern
- `main` branch: working branch for new content
- Version branches (`0.7.0`, `0.7.1`, etc.): created by CI on each release
- Terraform refs on version branches still point to `ref=main` (terraform doesn't change per release)

### CI Integration

Add this to your release workflow:

```yaml
- name: Create docs branch for version
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    VERSION="${{ steps.version.outputs.version }}"
    git clone https://x-access-token:${GH_TOKEN}@github.com/${{ github.repository_owner }}/my-docs.git /tmp/docs
    cd /tmp/docs
    git config user.email "ci@myapp.dev"
    git config user.name "CI"
    git checkout -b "${VERSION}"
    git push origin "${VERSION}"
```

## Terraform Module Repo Structure

Keep Terraform modules in a separate repo (not the docs repo). Structure:

```
├── modules/
│   ├── aws/
│   │   ├── main.tf        # EKS + RDS + ElastiCache + VPC
│   │   ├── variables.tf   # All input variables with descriptions
│   │   └── outputs.tf     # Cluster endpoint, DB endpoint, helm command
│   ├── gcp/
│   │   ├── main.tf        # GKE + Cloud SQL + Memorystore + VPC
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── azure/
│       ├── main.tf        # AKS + Azure DB + Azure Cache + VNet
│       ├── variables.tf
│       └── outputs.tf
```

### What the Modules Should Provision

Each module should create the infrastructure that replaces embedded subcharts:

| Embedded Component | AWS Replacement | GCP Replacement | Azure Replacement |
|-------------------|-----------------|-----------------|-------------------|
| PostgreSQL subchart | RDS | Cloud SQL | Azure DB for PostgreSQL Flexible |
| Redis subchart | ElastiCache | Memorystore | Azure Cache for Redis |
| Kubernetes cluster | EKS | GKE | AKS |
| Networking | VPC + subnets + SGs | VPC + subnets + firewall | VNet + subnets + NSGs |

### Output a Helm Install Command

The most useful output is a pre-filled `helm install` command:

```hcl
output "helm_install_command" {
  value = <<-EOT
    helm install my-app \
      oci://registry.replicated.com/my-app/stable/my-app \
      --set postgres.enabled=false \
      --set externalDatabase.host=${aws_db_instance.app.address} \
      --set redis.enabled=false \
      --set externalRedis.host=${aws_elasticache_replication_group.app.primary_endpoint_address}
  EOT
}
```

## Entitlement Fields for EP Visibility

Create these license fields in the vendor portal to gate EP content:

| Field | Type | Purpose |
|-------|------|---------|
| `isAWSEnabled` | Boolean | Show AWS terraform module |
| `isGCPEnabled` | Boolean | Show GCP terraform module |
| `isAzureEnabled` | Boolean | Show Azure terraform module |
| `isHAEnabled` | Boolean | Show HA configuration sections |

These are in addition to the built-in `isEmbeddedClusterDownloadEnabled`, `isHelmInstallEnabled`, and `isAirgapSupported` fields.

## Linking the Docs Repo

In the vendor portal:
1. Go to your app settings or Enterprise Portal configuration
2. Set the GitHub Collab Repo to your docs repo (e.g. `myorg/my-docs`)
3. The GitHub App integration must have access to the repo
4. Content is synced from the branch matching the customer's release version
