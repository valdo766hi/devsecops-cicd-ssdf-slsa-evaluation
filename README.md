# DevSecOps CI/CD - SSDF & SLSA Evaluation

A GitOps-based Kubernetes infrastructure repository for evaluating **SSDF (Secure Software Development Framework)** and **SLSA (Supply chain Levels for Software Artifacts)** compliance in CI/CD pipelines.

## Overview

This repository implements Infrastructure-as-Code (IaC) using the GitOps "app-of-apps" pattern, where ArgoCD declaratively manages all cluster components from Git as the single source of truth.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Git Repository                           │
│                  (Single Source of Truth)                       │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                         ArgoCD                                  │
│                    (GitOps Controller)                          │
│         ┌───────────────┴───────────────┐                       │
│         ▼                               ▼                       │
│   App-of-Apps                    Self-Management                │
│   (argocd-apps)                  (argocd)                       │
└─────────────────────────┬───────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  Namespaces │   │ Gateway API │   │    Istio    │
│  (Wave 0)   │   │  (Wave 2)   │   │ (Wave 2-3)  │
└─────────────┘   └─────────────┘   └──────┬──────┘
                                           │
                                           ▼
                                   ┌─────────────┐
                                   │   Gateway   │
                                   │  (Ingress)  │
                                   └──────┬──────┘
                                          │
                                          ▼
                                *.skripsi.oryzasa.site
```

## Tech Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| **ArgoCD** | v9.3.5 | GitOps continuous deployment |
| **Istio** | v1.28.3 | Service mesh & Gateway API controller |
| **Gateway API** | v1.4.1 | Kubernetes-native advanced routing |
| **Kustomize** | - | YAML configuration management |
| **Helm** | - | Kubernetes package manager |

## Directory Structure

```
.
├── argocd/                      # ArgoCD configuration
│   ├── applications/            # ArgoCD Application manifests
│   │   ├── namespaces.yaml      # Wave 0: Namespace creation
│   │   ├── argocd.yaml          # Wave 1: ArgoCD self-deployment
│   │   ├── argocd-apps.yaml     # Wave 1: App-of-apps orchestrator
│   │   ├── gateway-api.yaml     # Wave 2: Gateway API CRDs
│   │   └── istio.yaml           # Wave 2-3: Istio components
│   ├── projects/                # ArgoCD AppProject definitions
│   │   └── devsecops-skripsi.yaml
│   └── kustomization.yaml
│
├── deploy/                      # Deployment configurations
│   ├── manifests/               # Kubernetes manifests (Kustomize)
│   │   ├── namespaces/          # Namespace definitions
│   │   ├── argocd/              # ArgoCD HTTPRoute
│   │   └── istio/               # Istio Gateway configuration
│   └── helm-values/             # Helm chart value overrides
│       ├── argocd/              # ArgoCD values
│       └── istio/               # Istio component values
│           ├── base/
│           ├── istiod/
│           └── gateway/
│
├── crds/                        # Custom Resource Definitions
│   └── gateway-api/             # Kubernetes Gateway API CRDs
│       ├── standard/            # Production-ready CRDs
│       └── experimental/        # Beta feature CRDs
│
├── charts/                      # Helm charts
│   └── crapi/                   # OWASP crAPI (planned)
│
├── github/                      # GitHub configurations (planned)
│
└── docs/                        # Documentation
    ├── ARCHITECTURE.md
    └── EXPERIMENT.md
```

## Deployment Flow

Applications are deployed in **sync waves** to ensure proper dependency ordering:

| Wave | Application | Description |
|------|-------------|-------------|
| 0 | Namespaces | Creates required namespaces (istio-system, etc.) |
| 1 | ArgoCD | Self-bootstrapping ArgoCD deployment |
| 1 | ArgoCD Apps | App-of-apps pattern orchestrator |
| 2 | Gateway API | Installs Gateway API CRDs |
| 2 | Istio Base | Core Istio components |
| 3 | Istiod | Istio control plane |
| 3 | Istio Gateway | Ingress gateway controller |

## Network Configuration

### Domain
- **Wildcard**: `*.skripsi.oryzasa.site`
- **ArgoCD UI**: `argocd.skripsi.oryzasa.site`

### Ingress Architecture
```
External Traffic
       │
       ▼ (NodePort 30080)
┌─────────────────┐
│  Istio Gateway  │
│  (Gateway API)  │
└────────┬────────┘
         │
         ▼ (HTTPRoute)
┌─────────────────┐
│    Services     │
│ (ArgoCD, etc.)  │
└─────────────────┘
```

## Getting Started

### Prerequisites
- Kubernetes cluster (v1.28+)
- kubectl configured
- ArgoCD CLI (optional)

### Bootstrap Installation

1. **Install ArgoCD**
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. **Apply the App-of-Apps**
   ```bash
   kubectl apply -f argocd/applications/argocd-apps.yaml
   ```

3. **Access ArgoCD UI**
   ```bash
   # Get initial admin password
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

   # Access via configured domain or port-forward
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```

## Configuration

### ArgoCD Settings
- **Sync Policy**: Automated with prune and self-heal
- **Retry**: Exponential backoff (max 5 minutes, 5 retries)
- **HTTP Mode**: Insecure (no TLS termination at ArgoCD)

### Istio Settings
- **Replicas**: 1 (development/test configuration)
- **Service Type**: NodePort
- **Gateway Class**: `istio`

## Security Considerations

This repository implements DevSecOps practices aligned with:

### SSDF (Secure Software Development Framework)
- Version-controlled infrastructure definitions
- Declarative, auditable configuration changes
- Automated deployment reducing manual intervention

### SLSA (Supply chain Levels for Software Artifacts)
- Git as single source of truth
- Automated build and deployment pipeline
- Provenance through Git commit history

## Development Status

| Feature | Status |
|---------|--------|
| ArgoCD GitOps | Implemented |
| Istio Service Mesh | Implemented |
| Gateway API | Implemented |
| GitHub Actions CI/CD | Planned |
| crAPI Test Application | Planned |
| SLSA Provenance | Planned |
| Documentation | In Progress |

## Contributing

1. Fork the repository
2. Create a feature branch
3. Commit changes with descriptive messages
4. Submit a pull request

## License

This project is part of a thesis research on DevSecOps CI/CD pipeline security evaluation.

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Istio Documentation](https://istio.io/latest/docs/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [NIST SSDF](https://csrc.nist.gov/publications/detail/sp/800-218/final)
- [SLSA Framework](https://slsa.dev/)
