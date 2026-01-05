# ArgoCD App of Apps Demo

This repository demonstrates the **App of Apps** pattern with ArgoCD for OpenShift deployments.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Hub Cluster                                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    OpenShift GitOps                          │    │
│  │                                                              │    │
│  │  demo-apps-root (Application)                                │    │
│  │       │                                                      │    │
│  │       ├── Creates: AppProject (demo-apps)                    │    │
│  │       ├── Creates: Application (httpd-demo)  ──────┐        │    │
│  │       ├── Creates: Application (pacman)      ──────┼────────┼────┼──► CT1 Cluster
│  │       └── Creates: Application (guestbook)   ──────┘        │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Manages
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          CT1 Cluster                                 │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │   httpd-demo    │  │     pacman      │  │    guestbook    │     │
│  │   namespace     │  │   namespace     │  │   namespace     │     │
│  │                 │  │                 │  │                 │     │
│  │  - Deployment   │  │  - MongoDB      │  │  - Redis Leader │     │
│  │  - Service      │  │  - Pacman App   │  │  - Redis Follow │     │
│  │  - Route        │  │  - Service      │  │  - Frontend     │     │
│  │                 │  │  - Route        │  │  - Route        │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

## Repository Structure

```
argo-apps-demo/
├── base/                           # App of Apps root
│   ├── kustomization.yaml          # Kustomize configuration
│   ├── appproject.yaml             # ArgoCD AppProject
│   ├── app-httpd-demo.yaml         # Application CR for httpd
│   ├── app-pacman.yaml             # Application CR for pacman
│   └── app-guestbook.yaml          # Application CR for guestbook
│
├── apps/                           # Application manifests
│   ├── httpd-demo/                 # Apache HTTPD demo
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── route.yaml
│   │
│   ├── pacman/                     # Pacman game demo
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── mongodb-*.yaml          # MongoDB backend
│   │   └── pacman-*.yaml           # Pacman frontend
│   │
│   └── guestbook/                  # Classic K8s demo
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── redis-leader-*.yaml     # Redis leader
│       ├── redis-follower-*.yaml   # Redis followers
│       └── guestbook-*.yaml        # Guestbook frontend
│
└── README.md
```

## Demo Applications

### 1. HTTPD Demo
Simple Apache web server using Red Hat's UBI image.
- **Namespace**: `httpd-demo`
- **Replicas**: 2
- **Image**: `registry.redhat.io/rhel8/httpd-24:latest`

### 2. Pacman
Classic arcade game with MongoDB backend for high scores.
- **Namespace**: `pacman`
- **Components**: MongoDB + Node.js frontend
- **Image**: `quay.io/jpacker/nodejs-pacman-app:latest`

### 3. Guestbook
Classic Kubernetes demo with Redis backend.
- **Namespace**: `guestbook`
- **Components**: Redis leader + followers + PHP frontend
- **Image**: `gcr.io/google_samples/gb-frontend:v5`

## Deployment

### Prerequisites
1. Hub cluster with OpenShift GitOps installed
2. CT1 cluster registered in Hub ArgoCD
3. Network connectivity between clusters

### Deploy via Hub ArgoCD

The root application should be deployed from the Hub cluster:

```bash
# Apply the root application from the argo-demo repo
oc apply -f clusters/hub/demo-apps-root-application.yaml
```

Or add to the hub's kustomization.yaml:

```yaml
resources:
  - demo-apps-root-application.yaml
```

### Verify Deployment

```bash
# Check Application status in Hub
oc get applications -n openshift-gitops

# Check deployed resources on CT1
oc get all -n httpd-demo
oc get all -n pacman
oc get all -n guestbook

# Get application routes
oc get routes -n httpd-demo
oc get routes -n pacman
oc get routes -n guestbook
```

## App of Apps Pattern Benefits

1. **Single Point of Management**: Deploy multiple applications with one root Application
2. **GitOps Native**: All configuration stored in Git
3. **Scalable**: Easy to add new applications
4. **Consistent**: All apps follow same deployment patterns
5. **Self-Healing**: ArgoCD automatically reconciles drift

## Customization

### Adding New Applications

1. Create application manifests in `apps/<app-name>/`
2. Add Application CR in `base/app-<app-name>.yaml`
3. Update `base/kustomization.yaml` resources
4. Update `base/appproject.yaml` destinations if needed

### Environment-Specific Overlays

Create overlays for different environments:

```
overlays/
├── dev/
│   └── kustomization.yaml
├── staging/
│   └── kustomization.yaml
└── prod/
    └── kustomization.yaml
```

## Troubleshooting

### Common Issues

1. **Application stuck in Unknown state**
   - Verify cluster connectivity
   - Check ArgoCD has access to target cluster

2. **Sync failed - namespace not found**
   - Ensure `CreateNamespace=true` in syncOptions

3. **Image pull errors**
   - Verify image registry access
   - Check pull secrets if using private registry

### Useful Commands

```bash
# Force sync an application
argocd app sync demo-apps-root --prune

# Get application details
argocd app get httpd-demo

# View sync history
argocd app history pacman

# Refresh application
argocd app get guestbook --refresh
```

## License

Apache 2.0

