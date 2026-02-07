# Architecture Documentation

## System Architecture

### GitOps Control Plane

```
┌─────────────────────────────────────────────────────────────────┐
│                        Git Repository                           │
│                  (Single Source of Truth)                       │
│                                                                 │
│  deploy/manifests/                                              │
│  ├── <component>/                                               │
│  │   ├── app.yaml          # ArgoCD Application metadata        │
│  │   └── values/                                                  │
│  │       └── values.yaml   # Helm configuration                 │
│  └── ...                                                        │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ArgoCD ApplicationSet                        │
│                                                                 │
│  Generator: Git (deploy/manifests/**/app.yaml)                  │
│  └── Discovers all app.yaml files automatically                 │
│                                                                 │
│  Template:                                                      │
│  ├── name: {{ .name }}                                          │
│  ├── namespace: {{ .namespace }}                                │
│  ├── sources: {{ .sources }}                                    │
│  └── syncPolicy: Automated + SelfHeal                           │
└─────────────────────────────────────────────────────────────────┘
```

### Component Dependencies

```
Wave 0: Namespaces
    │
    ▼
Wave 1: ArgoCD + Gateway API
    │
    ├──────────┬──────────┐
    ▼          ▼          ▼
Wave 2:  cert-manager    Istio
    │                    │
    ▼                    ▼
Wave 3:              Rook-Ceph
    │                    │
    ├──────────┬──────────┼──────────┐
    ▼          ▼          ▼          ▼
Wave 4:   OpenBao  External Secrets  Trivy Operator
```

### Storage Architecture

**Rook-Ceph Block Storage:**
- **Use case**: Persistent volumes for stateful workloads
- **StorageClass**: `rook-ceph-block`
- **Provisioner**: `rook-ceph.rbd.csi.ceph.com`
- **Backend**: Ceph RBD (RADOS Block Device)
- **Replication**: 1 replica (single-node setup)

**Used by:**
- OpenBao (vault data persistence)
- Any PVC using `rook-ceph-block` storage class

### Secrets Management Architecture

**OpenBao (HashiCorp Vault alternative):**
- **Deployment**: Single-node (replicas: 1)
- **Storage**: Rook-Ceph PVC
- **Access**: Internal cluster access via Kubernetes auth

**External Secrets Operator:**
- **Purpose**: Sync secrets from external stores (OpenBao) to Kubernetes
- **Pattern**: ClusterSecretStore → ExternalSecret → Kubernetes Secret
- **Refresh**: Automatic (1m interval)

**Secret Flow:**
```
OpenBao (external store)
    │
    ▼ (ClusterSecretStore)
External Secrets Operator
    │
    ▼ (ExternalSecret CR)
Kubernetes Secret
    │
    ▼ (mounted as volume/env)
Application Pod
```

### Security Scanning Architecture

**Trivy Operator:**
- **Mode**: ClientServer (uses built-in Trivy server)
- **Scans**:
  - Vulnerability scanning (CVE database)
  - Configuration audit (security misconfigs)
  - RBAC assessment (overprivileged roles)
  - Exposed secrets detection
- **Reports**: Stored as Kubernetes CRDs (VulnerabilityReport, ConfigAuditReport, etc.)

**Scan Flow:**
```
Trivy Operator
    │
    ├── Discovers workloads
    │
    ├── Creates scan jobs (one per unique image)
    │
    ├── Jobs pull Trivy server (cached vuln DB)
    │
    └── Generates reports as CRDs
```

### CI/CD Reporting Surfaces

This repository uses GitHub Actions for CI/CD security checks. Results are surfaced in three places:

- **GitHub Actions logs**: human-readable tables (Trivy, Kyverno, etc.)
- **GitHub Security 3 Code scanning**: SARIF uploads for scan tracking
- **Workflow artifacts**: downloadable report files for audit/research evidence

### Network Architecture

**Gateway API (Istio implementation):**
- **Gateway**: `gateway` in `istio-system`
- **Protocols**: HTTP (80) + HTTPS (443 with TLS)
- **TLS**: Automated via cert-manager + Let's Encrypt

**HTTPRoute Pattern:**
```yaml
ParentRef: Gateway (istio-system/gateway)
Hostname: <service>.skripsi.oryzasa.site
Rules:
  - HTTP → HTTPS redirect (port 80)
  - HTTPS → Service backend (port <app-port>)
```

**Current Routes:**
- `argocd.skripsi.oryzasa.site` → ArgoCD server (port 80)
- `ceph.skripsi.oryzasa.site` → Ceph dashboard (port 7000)

## Data Flow

### Application Deployment Flow

1. **Developer** commits changes to `app.yaml` or `values.yaml`
2. **Git push** triggers ArgoCD webhook (or 3m poll)
3. **ApplicationSet** discovers new/changed `app.yaml`
4. **ArgoCD** generates Application CR from template
5. **Sync** begins based on sources (Helm, Kustomize, plain YAML)
6. **Resources** created in target namespace
7. **Health checks** verify deployment success

### Certificate Flow

1. **HTTPRoute** created with hostname
2. **Gateway** sees new route, triggers certificate request
3. **cert-manager** creates Certificate CR
4. **Let's Encrypt** challenge initiated (DNS-01 via Cloudflare)
5. **Certificate** issued and stored as TLS secret
6. **Gateway** uses secret for HTTPS termination

## Resource Allocation

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| ArgoCD | 100m | 500m | 256Mi | 512Mi |
| Istiod | 500m | 2000m | 512Mi | 2Gi |
| Gateway | 100m | 500m | 128Mi | 256Mi |
| cert-manager | 10m | 100m | 32Mi | 128Mi |
| Rook-Ceph Operator | 100m | 500m | 256Mi | 512Mi |
| Rook-Ceph Mon | 1000m | 2000m | 1Gi | 2Gi |
| Rook-Ceph MGR | 500m | 1000m | 512Mi | 1Gi |
| Rook-Ceph OSD | 1000m | - | 4Gi | 4Gi |
| OpenBao | 100m | 500m | 256Mi | 512Mi |
| External Secrets | 100m | 500m | 128Mi | 256Mi |
| Trivy Operator | 100m | 500m | 100Mi | 500Mi |

## Scalability Considerations

**Current Limitations (Single Node):**
- All pods scheduled on one node
- No real high availability (single points of failure)
- Ceph replication factor 1 (no redundancy)

**Future Scaling:**
- Add second node → Enable Ceph replication factor 2
- Enable OpenBao HA mode (replicas: 3)
- Istio gateway can scale horizontally
- ArgoCD supports HA mode with multiple replicas

## Monitoring Points

**Key Metrics to Monitor:**
1. **ArgoCD**: Sync status, app health, reconciliation errors
2. **Istio**: Request rates, error rates, latency
3. **Ceph**: OSD health, PG status, capacity usage
4. **cert-manager**: Certificate expiry, renewal failures
5. **Trivy**: Critical/high CVE count, scan job failures

**Alert Recommendations:**
- Ceph health not OK
- Certificate expiring in < 7 days
- Critical CVE found in scanned images
- ArgoCD app sync failed
