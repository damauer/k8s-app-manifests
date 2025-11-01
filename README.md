# Kubernetes Application Manifests

GitOps repository for managing Kubernetes applications across DEV and PROD environments using ArgoCD.

## Repository Structure

```
k8s-app-manifests/
├── apps/
│   ├── dev/              # DEV environment application manifests
│   │   └── <app-name>/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── ...
│   └── prod/             # PROD environment application manifests
│       └── <app-name>/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ...
├── argocd/
│   ├── projects/         # ArgoCD project definitions
│   │   ├── dev-project.yaml
│   │   └── prod-project.yaml
│   └── applications/     # ArgoCD application definitions
│       ├── dev/
│       │   └── <app>-dev.yaml
│       └── prod/
│           └── <app>-prod.yaml
└── .github/
    └── workflows/        # CI workflows for validation
```

## Architecture

### Environments

- **DEV Cluster** (Go-based)
  - Fast deployment and iterations
  - Runs on local Multipass VMs
  - Hosts ArgoCD for managing both clusters
  - Immediate feedback on changes

- **PROD Cluster** (Rust-based)
  - Optimized for stability and performance
  - Runs on local Multipass VMs
  - Managed by ArgoCD on DEV cluster
  - Changes promoted from DEV after validation

## GitOps Workflow

### Development Flow

1. **Create/Update Application**
   ```bash
   # Edit manifests in apps/dev/
   git add apps/dev/<app-name>/
   git commit -m "Update <app-name> in DEV"
   git push
   ```

2. **ArgoCD Auto-Sync**
   - ArgoCD detects changes in Git
   - Automatically syncs to DEV cluster
   - Monitor deployment in ArgoCD UI

3. **Validate in DEV**
   ```bash
   kubectl get pods -n <namespace>
   kubectl logs -n <namespace> <pod-name>
   ```

4. **Promote to PROD**
   ```bash
   # Copy validated manifests to prod/
   cp -r apps/dev/<app-name> apps/prod/
   # Adjust environment-specific values
   # Commit and push
   git add apps/prod/<app-name>/
   git commit -m "Promote <app-name> to PROD"
   git push
   ```

## Getting Started

### Prerequisites

- DEV cluster deployed and running (see [k8s-infrastructure](https://github.com/damauer/k8s-infrastructure))
- PROD cluster deployed and running
- ArgoCD installed on DEV cluster
- kubectl configured to access both clusters

### Access ArgoCD

```bash
# ArgoCD is exposed on DEV cluster
# Get the NodePort
kubectl get svc -n argocd argocd-server

# Access at: http://<dev-control-plane-ip>:<nodeport>

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Application Management

### Creating a New Application

1. **Create manifest directory**
   ```bash
   mkdir -p apps/dev/<app-name>
   ```

2. **Add Kubernetes manifests**
   - deployment.yaml
   - service.yaml
   - configmap.yaml (optional)
   - ingress.yaml (optional)

3. **Create ArgoCD Application**
   ```yaml
   # argocd/applications/dev/<app-name>-dev.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: <app-name>-dev
     namespace: argocd
   spec:
     project: dev-project
     source:
       repoURL: https://github.com/damauer/k8s-app-manifests
       targetRevision: main
       path: apps/dev/<app-name>
     destination:
       server: https://kubernetes.default.svc
       namespace: default
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```

4. **Commit and push**
   ```bash
   git add apps/dev/<app-name>/ argocd/applications/dev/
   git commit -m "Add <app-name> to DEV"
   git push
   ```

### Updating an Application

1. Edit manifests in `apps/dev/<app-name>/`
2. Commit and push changes
3. ArgoCD automatically syncs (if auto-sync enabled)
4. Or manually sync via ArgoCD UI/CLI

### Promoting to PROD

1. **Copy manifests**
   ```bash
   cp -r apps/dev/<app-name> apps/prod/
   ```

2. **Adjust for PROD**
   - Update resource limits
   - Change replica counts
   - Adjust environment variables
   - Update image tags to stable versions

3. **Create PROD ArgoCD Application**
   ```yaml
   # argocd/applications/prod/<app-name>-prod.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: <app-name>-prod
     namespace: argocd
   spec:
     project: prod-project
     source:
       repoURL: https://github.com/damauer/k8s-app-manifests
       targetRevision: main
       path: apps/prod/<app-name>
     destination:
       server: https://<prod-cluster-url>
       namespace: default
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```

4. **Commit and push**
   ```bash
   git add apps/prod/<app-name>/ argocd/applications/prod/
   git commit -m "Promote <app-name> to PROD"
   git push
   ```

## CI/CD Integration

### GitHub Actions

This repository includes CI workflows that:
- Validate YAML syntax
- Check Kubernetes manifest structure
- Run policy checks (if configured)
- Prevent invalid manifests from merging

### Pull Request Workflow

1. Create feature branch
2. Make changes to manifests
3. Open pull request
4. CI runs validation
5. Manual review and approval
6. Merge to main
7. ArgoCD auto-syncs to cluster

## Environment Differences

### DEV
- Lower resource limits
- Faster sync intervals
- Auto-pruning enabled
- Self-healing enabled
- Latest image tags

### PROD
- Higher resource limits
- Manual sync option available
- Careful pruning policies
- Stable image tags
- Additional health checks

## Monitoring

### ArgoCD Dashboard

Access the ArgoCD UI to:
- View application status
- See sync history
- Trigger manual syncs
- View logs and events
- Compare Git vs Live state

### Application Health

```bash
# Check application health in ArgoCD
argocd app get <app-name>-dev
argocd app get <app-name>-prod

# View sync status
argocd app sync <app-name>-dev
argocd app sync <app-name>-prod
```

## Troubleshooting

### Application Not Syncing

```bash
# Check ArgoCD application status
argocd app get <app-name>-dev

# View detailed sync errors
kubectl describe application <app-name>-dev -n argocd

# Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### Manifest Errors

```bash
# Validate YAML locally
kubectl apply --dry-run=client -f apps/dev/<app-name>/

# Check manifest with kubeval
kubeval apps/dev/<app-name>/*.yaml
```

## Best Practices

1. **Always test in DEV first**
2. **Use semantic versioning for images**
3. **Keep DEV and PROD manifests similar** (only environment-specific differences)
4. **Use ConfigMaps and Secrets** for configuration
5. **Add resource requests and limits**
6. **Include health checks** (readiness/liveness probes)
7. **Use meaningful commit messages**
8. **Tag releases** for PROD deployments

## Security

- Never commit secrets or credentials
- Use Kubernetes Secrets or external secret management
- Review changes in pull requests
- Use RBAC to control access
- Monitor for drift between Git and cluster state

## Related Repositories

- [k8s-infrastructure](https://github.com/damauer/k8s-infrastructure) - Cluster installation and management
