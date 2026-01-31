# DevSecOps CI/CD - SSDF & SLSA Evaluation

A GitOps-based Kubernetes infrastructure repository for evaluating **SSDF (Secure Software Development Framework)** and **SLSA (Supply chain Levels for Software Artifacts)** compliance in CI/CD pipelines.

## Overview

This repository implements Infrastructure-as-Code (IaC) using **ArgoCD ApplicationSet** pattern, where ArgoCD automatically discovers and manages applications from `app.yaml` files distributed across the repository. Git serves as the single source of truth for all cluster components.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Git Repository                           │
│                  (Single Source of Truth)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  app.yaml   │  │  app.yaml   │  │  app.yaml   │             │
│  │  + values   │  │  + values   │  │  + values   │             │
│  │  (ArgoCD    │  │  (Istio)    │  │(cert-manager│             │
│  │  manifests) │  │             │  │             │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                         ArgoCD                                  │
│              (GitOps Controller + ApplicationSet)               │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │           infrastructure-appset                         │   │
│   │   Generator: Git files (deploy/manifests/**/app.yaml)   │   │
│   └────────────────────────┬────────────────────────────────┘   │
└────────────────────────────┼────────────────────────────────────┘
                             │
     ┌───────────────────────┼───────────────────────┐
     ▼                       ▼                       ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Namespaces  │    │ Gateway API │    │    Istio    │
│  (Wave 0)   │    │  (Wave 1)   │    │ (Wave 2-3)  │
└─────────────┘    └─────────────┘    └──────┬──────┘
                                              │
┌─────────────┐    ┌─────────────┐    ┌──────▼──────┐
│  Rook-Ceph  │    │   OpenBao   │    │   Gateway   │
│  (Storage)  │    │  (Secrets)  │    │  (Ingress)  │
└─────────────┘    └─────────────┘    └─────────────┘
                                              │
┌─────────────┐    ┌─────────────┐    ┌──────▼──────┐
│External     │    │   Trivy     │    │  HTTPRoute  │
│  Secrets    │    │  Operator   │    │   Rules     │
└─────────────┘    └─────────────┘    └─────────────┘
                                              │
                                              ▼
                                    *.skripsi.oryzasa.site
```

## Tech Stack

| Component | Purpose | Status |
|-----------|---------|--------|
| **ArgoCD** | GitOps continuous deployment | ✅ Active |
| **Istio** | Service mesh & Gateway API controller | ✅ Active |
| **Gateway API** | Kubernetes-native advanced routing | ✅ Active |
| **cert-manager** | Automated TLS certificate management | ✅ Active |
| **Rook-Ceph** | Distributed storage (block storage) | ✅ Active |
| **OpenBao** | Secrets management (Vault alternative) | ✅ Active |
| **External Secrets** | Kubernetes external secrets integration | ✅ Active |
| **Trivy Operator** | Container vulnerability scanning | ✅ Active |

## Repository Structure

```
.
├── argocd/                      # ArgoCD configuration
│   ├── appsets/                 # ApplicationSet definitions
│   │   └── applicationsets.yaml # Infrastructure ApplicationSet
│   └── kustomization.yaml       # Kustomize entry point
│
├── deploy/                      # Application configurations
│   └── manifests/               # Kubernetes manifests
│       ├── namespaces/          # Namespace definitions
│       │   ├── app.yaml         # ArgoCD Application metadata
│       │   └── *.yaml           # Namespace manifests
│       ├── argocd/              # ArgoCD HTTPRoute
│       │   ├── app.yaml
│       │   └── httproute.yaml
│       ├── gateway-api/         # Gateway API CRDs
│       │   └── app.yaml
│       ├── cert-manager/        # TLS certificate management
│       │   ├── app.yaml
│       │   └── clusterissuer.yaml
│       ├── istio/               # Service mesh components
│       │   ├── app.yaml
│       │   ├── base/
│       │   ├── istiod/
│       │   └── gateway/
│       ├── rook-ceph/           # Storage (Ceph cluster)
│       │   ├── app.yaml
│       │   ├── operator/        # Rook operator values
│       │   └── cluster/         # Ceph cluster values
│       ├── openbao/             # Secrets management
│       │   ├── app.yaml
│       │   └── values/
│       ├── external-secrets/    # External secrets operator
│       │   ├── app.yaml
│       │   ├── values/
│       │   └── test-external-secret.yaml
│       └── trivy-operator/      # Vulnerability scanning
│           ├── app.yaml
│           └── values/
│
├── crds/                        # Custom Resource Definitions
│   └── gateway-api/             # Gateway API CRDs
│       ├── standard/
│       └── experimental/
│
└── docs/                        # Documentation
    ├── ARCHITECTURE.md
    └── EXPERIMENT.md
```

## Application Pattern

Each component follows a consistent pattern for ArgoCD discovery:

### 1. `app.yaml` - ArgoCD Application Metadata

Located at `deploy/manifests/<component>/app.yaml`:

```yaml
name: <component-name>
project: devsecops-skripsi
namespace: <target-namespace>
sync:
  automated: true
  prune: false
  selfHeal: true
  createNamespace: true
labels:
  app.kubernetes.io/part-of: infrastructure
  app.kubernetes.io/component: <component-name>
sources:
  # Helm chart source (if applicable)
  - repoURL: https://charts.example.com
    chart: <chart-name>
    targetRevision: <version>
    helm:
      valueFiles:
        - $values/deploy/manifests/<component>/values/values.yaml
  # Git ref for values
  - repoURL: git@github.com:valdo766hi/devsecops-cicd-ssdf-slsa-evaluation.git
    targetRevision: HEAD
    ref: values
  # Additional manifests path
  - repoURL: git@github.com:valdo766hi/devsecops-cicd-ssdf-slsa-evaluation.git
    targetRevision: HEAD
    path: deploy/manifests/<component>
```

### 2. `values.yaml` - Helm Configuration

Located at `deploy/manifests/<component>/values/values.yaml`:

```yaml
# Component-specific Helm values
replicas: 1
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

## Deployment Flow

Components are deployed in **sync waves** to ensure proper dependency ordering:

| Wave | Component | Dependencies | Purpose |
|------|-----------|--------------|---------|
| 0 | Namespaces | None | Creates all required namespaces |
| 1 | ArgoCD | Namespaces | Self-bootstrapping |
| 1 | Gateway API | Namespaces | CRDs for routing |
| 2 | cert-manager | Gateway API | TLS certificate automation |
| 2 | Istio | Gateway API | Service mesh |
| 3 | Rook-Ceph | Istio | Block storage |
| 3 | OpenBao | Rook-Ceph | Secrets management |
| 3 | External Secrets | OpenBao | External secrets integration |
| 4 | Trivy Operator | Rook-Ceph | Vulnerability scanning |

## Network Configuration

### Domains
- **Wildcard**: `*.skripsi.oryzasa.site`
- **ArgoCD UI**: `argocd.skripsi.oryzasa.site`
- **Ceph Dashboard**: `ceph.skripsi.oryzasa.site`
- **OpenBao**: `vault.skripsi.oryzasa.site` (if enabled)

### Ingress Architecture
```
External Traffic
       │
       ▼ (NodePort 30080)
┌─────────────────┐
│  Istio Gateway  │
│  (Gateway API)  │
│  TLS: Let's     │
│  Encrypt        │
└────────┬────────┘
         │
         ▼ (HTTPRoute)
┌─────────────────┐
│    Services     │
│ (ArgoCD, Ceph,  │
│  etc.)          │
└─────────────────┘
```

## Getting Started

### Prerequisites
- Kubernetes cluster (v1.28+)
- kubectl configured
- ArgoCD CLI (optional)

### Bootstrap Installation

1. **Install ArgoCD manually first:**
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. **Apply the ApplicationSet:**
   ```bash
   kubectl apply -f argocd/appsets/applicationsets.yaml
   ```

3. **Access ArgoCD UI:**
   ```bash
   # Get initial admin password
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
   
   # Port-forward for access
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```

4. **Add Git repository to ArgoCD:**
   Configure via UI or CLI with SSH key authentication.

## Adding New Components

To add a new infrastructure component:

1. **Create directory structure:**
   ```bash
   mkdir -p deploy/manifests/<component>/values
   ```

2. **Create `app.yaml`:**
   ```yaml
   name: <component>
   project: devsecops-skripsi
   namespace: <namespace>
   sync:
     automated: true
     prune: false
     selfHeal: true
     createNamespace: true
   labels:
     app.kubernetes.io/part-of: infrastructure
     app.kubernetes.io/component: <component>
   sources:
     - repoURL: <chart-repo>
       chart: <chart-name>
       targetRevision: <version>
       helm:
         valueFiles:
           - $values/deploy/manifests/<component>/values/values.yaml
     - repoURL: git@github.com:valdo766hi/devsecops-cicd-ssdf-slsa-evaluation.git
       targetRevision: HEAD
       ref: values
   ```

3. **Create `values.yaml` (if using Helm):**
   ```yaml
   # Component-specific configuration
   ```

4. **Trigger ArgoCD sync:**
   The ApplicationSet will automatically discover the new `app.yaml` within 3 minutes, or trigger manually:
   ```bash
   kubectl -n argocd annotate applicationset infrastructure-appset \
     argocd.argoproj.io/refresh=hard --overwrite
   ```

## Configuration

### ArgoCD ApplicationSet
- **Generator**: Git files matching `deploy/manifests/**/app.yaml`
- **Sync Policy**: Automated with self-heal
- **Retry**: Exponential backoff (max 5 minutes, 8 retries)
- **Namespace Creation**: Automatic via `CreateNamespace=true`

### Common Settings
- **Replicas**: 1 (development configuration)
- **Resources**: Configured per component
- **Storage**: Rook-Ceph block storage
- **Secrets**: OpenBao + External Secrets Operator

## Security Features

This repository implements DevSecOps practices:

### SSDF (Secure Software Development Framework)
- ✅ Version-controlled infrastructure definitions
- ✅ Declarative, auditable configuration changes
- ✅ Automated deployment reducing manual intervention
- ✅ Vulnerability scanning (Trivy Operator)
- ✅ Secrets management (OpenBao)

### SLSA (Supply chain Levels for Software Artifacts)
- ✅ Git as single source of truth
- ✅ Automated build and deployment pipeline
- ✅ Provenance through Git commit history
- ✅ Helm chart version pinning

## Development Status

| Feature | Status |
|---------|--------|
| ArgoCD ApplicationSet | ✅ Active |
| Istio Service Mesh | ✅ Active |
| Gateway API | ✅ Active |
| cert-manager | ✅ Active |
| Rook-Ceph Storage | ✅ Active |
| OpenBao Secrets | ✅ Active |
| External Secrets | ✅ Active |
| Trivy Operator | ✅ Active |
| GitHub Actions CI/CD | 📋 Planned |
| crAPI Test Application | 📋 Planned |
| SLSA Provenance | 📋 Planned |

## Troubleshooting

### Application not appearing in ArgoCD
- Check `app.yaml` syntax is valid
- Ensure file is at `deploy/manifests/<name>/app.yaml`
- Trigger ApplicationSet refresh: `kubectl -n argocd annotate applicationset infrastructure-appset argocd.argoproj.io/refresh=hard --overwrite`

### Sync failures
- Check component dependencies are met
- Verify namespace exists or `createNamespace: true` is set
- Review ArgoCD UI for specific error messages

### Scan job failures (Trivy)
- Normal during initial setup - some images may fail
- Check pod resource limits if OOM errors
- Verify registry access for private images

## Contributing

1. Fork the repository
2. Create a feature branch
3. Commit changes with descriptive messages
4. Test changes in your cluster
5. Submit a pull request

## License

This project is part of a thesis research on DevSecOps CI/CD pipeline security evaluation.

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
- [Istio Documentation](https://istio.io/latest/docs/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [Rook Ceph](https://rook.io/docs/rook/latest-release/Getting-Started/intro/)
- [OpenBao](https://openbao.org/docs/)
- [External Secrets](https://external-secrets.io/latest/)
- [Trivy Operator](https://aquasecurity.github.io/trivy-operator/latest/)
- [NIST SSDF](https://csrc.nist.gov/publications/detail/sp/800-218/final)
- [SLSA Framework](https://slsa.dev/)
