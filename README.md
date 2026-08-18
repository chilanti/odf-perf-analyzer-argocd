# analyze-argocd

GitOps-ready Kustomize project for the ODF Performance Dashboard, structured for ArgoCD management.

## Directory layout

```
analyze-argocd/
├── base/                          # Environment-agnostic manifests
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── serviceaccount.yaml        # opensearch ServiceAccount
│   ├── pvc.yaml                   # perf-results-pvc (prune-protected)
│   ├── deployment.yaml            # perf-dashboard Flask app
│   ├── service.yaml
│   ├── route.yaml                 # Edge-terminated HTTPS route
│   └── opensearch.yaml            # Service + StatefulSet
├── overlays/
│   └── dev/
│       ├── kustomization.yaml     # Sets namespace, image tag, storage patches
│       ├── patch-pvc-storage.yaml
│       └── patch-opensearch-storage.yaml
└── argocd/
    ├── perf-dashboard-app.yaml    # ArgoCD Application CR — apply once to the ArgoCD cluster
    └── opensearch-scc-crb.yaml   # ClusterRoleBinding: opensearch SA → anyuid SCC
```

## Before you apply

1. **Replace placeholders** in `overlays/prod/kustomization.yaml`:
   - `quay.io/YOUR_ORG/perf-dashboard` → your actual pre-built image reference
   - Pin `newTag` to a specific version rather than `latest` for production

2. **Replace placeholders** in `argocd/perf-dashboard-app.yaml`:
   - `repoURL` → your Git repository URL

3. **Storage classes** — the overlay defaults to:
   - `ocs-storagecluster-cephfs` for the dashboard PVC (`ReadWriteMany`)
   - `ocs-storagecluster-ceph-rbd` for the OpenSearch data volume
   Update these in the patch files if your cluster uses different class names.

## Deploying with ArgoCD

ArgoCD should point at the overlay, not the base:

```yaml
path: automation/odf-perf-next-gen/analyze-argocd/overlays/dev
```

Apply the `Application` CR once to bootstrap:

```bash
oc apply -f argocd/perf-dashboard-app.yaml
```

The `ClusterRoleBinding` (`opensearch-scc-crb.yaml`) grants the `opensearch` ServiceAccount
the `anyuid` SCC — required because OpenSearch runs as UID 1000. This replaces the
imperative `oc adm policy add-scc-to-user` call from the old `deploy.sh`.
The ArgoCD service account needs permission to create `ClusterRoleBindings`; grant it
cluster-admin or add the resource to a custom `AppProject` cluster resource whitelist.

## Local validation

Requires `kubectl` with the `kustomize` subcommand (v1.14+):

```bash
kubectl kustomize overlays/prod
```
