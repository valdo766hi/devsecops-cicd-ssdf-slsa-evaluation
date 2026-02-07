# Experiment Guide

## Quick Reference Commands

### ArgoCD Management

```bash
# Trigger ApplicationSet refresh (immediate sync)
kubectl -n argocd annotate applicationset infrastructure-appset \
  argocd.argoproj.io/refresh=hard --overwrite

# Check ArgoCD app status
kubectl -n argocd get applications

# View ArgoCD logs
kubectl -n argocd logs -l app.kubernetes.io/name=argocd-server

# Port-forward ArgoCD UI
kubectl -n argocd port-forward svc/argocd-server 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### Rook-Ceph Storage

```bash
# Check Ceph cluster health
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status

# List OSDs
kubectl -n rook-ceph get osd

# Check Ceph dashboard
# Access: https://ceph.skripsi.oryzasa.site
# Or port-forward:
kubectl -n rook-ceph port-forward svc/rook-ceph-mgr-dashboard 7000:7000

# Create test PVC
kubectl apply -f deploy/manifests/rook-ceph/pvc-test.yaml
kubectl -n rook-ceph get pvc rook-ceph-test-pvc

# Delete test PVC
kubectl delete -f deploy/manifests/rook-ceph/pvc-test.yaml
```

### Secrets Management (OpenBao)

```bash
# Check OpenBao status
kubectl -n openbao get pods

# Port-forward OpenBao UI
kubectl -n openbao port-forward svc/openbao 8200:8200

# View OpenBao logs
kubectl -n openbao logs -l app.kubernetes.io/name=openbao

# Check external secrets
kubectl -n external-secrets get externalsecrets
kubectl -n external-secrets get secretstores

# Test secret retrieval
kubectl -n external-secrets get secret skripsi-testing-secret -o yaml
```

### Security Scanning (Trivy)

```bash
# List all vulnerability reports
kubectl get vulnerabilityreports -A

# View specific report details
kubectl get vulnerabilityreport -n <namespace> <report-name> -o yaml

# Summary of critical/high vulnerabilities
kubectl get vulnerabilityreports -A -o custom-columns=' \
  NAMESPACE:.metadata.namespace, \
  NAME:.metadata.name, \
  CRITICAL:.report.summary.criticalCount, \
  HIGH:.report.summary.highCount, \
  MEDIUM:.report.summary.mediumCount'

# Watch for new scan reports
kubectl get vulnerabilityreports -A --watch

# Check scan job failures
kubectl -n trivy-system get jobs
kubectl -n trivy-system get pods | grep scan-vulnerabilityreport
```

### Certificate Management

```bash
# List all certificates
kubectl get certificates -A

# Check certificate status
kubectl describe certificate -n istio-system argocd-tls

# View cert-manager logs
kubectl -n cert-manager logs -l app.kubernetes.io/name=cert-manager

# Force certificate renewal
kubectl annotate certificate -n istio-system argocd-tls \
  cert-manager.io/retry-request="$(date +%s)"

# Check Cloudflare DNS challenge
kubectl -n cert-manager get challenges
kubectl -n cert-manager describe challenge <challenge-name>
```

## Common Tasks

### Adding a New Application

1. Create directory structure:
```bash
mkdir -p deploy/manifests/<app-name>/values
```

2. Create `deploy/manifests/<app-name>/app.yaml`:
```yaml
name: <app-name>
project: devsecops-skripsi
namespace: <namespace>
sync:
  automated: true
  prune: false
  selfHeal: true
  createNamespace: true
labels:
  app.kubernetes.io/part-of: infrastructure
  app.kubernetes.io/component: <app-name>
sources:
  - repoURL: <helm-chart-repo>
    chart: <chart-name>
    targetRevision: <version>
    helm:
      valueFiles:
        - $values/deploy/manifests/<app-name>/values/values.yaml
  - repoURL: git@github.com:valdo766hi/devsecops-cicd-ssdf-slsa-evaluation.git
    targetRevision: HEAD
    ref: values
```

3. Create `deploy/manifests/<app-name>/values/values.yaml`

4. Trigger sync:
```bash
kubectl -n argocd annotate applicationset infrastructure-appset \
  argocd.argoproj.io/refresh=hard --overwrite
```

### Adding HTTPRoute for a Service

Create `deploy/manifests/<app-name>/httproute.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app-name>
  namespace: <app-namespace>
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: gateway
      namespace: istio-system
      sectionName: http
  hostnames:
    - "<app-name>.skripsi.oryzasa.site"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app-name>-https
  namespace: <app-namespace>
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: gateway
      namespace: istio-system
      sectionName: https
  hostnames:
    - "<app-name>.skripsi.oryzasa.site"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - group: ''
          kind: Service
          name: <service-name>
          port: <service-port>
          weight: 1
```

## Troubleshooting

### ArgoCD Issues

**App not appearing:**
```bash
# Check if app.yaml is valid YAML
cat deploy/manifests/<app>/app.yaml | yq eval .

# Verify ApplicationSet can find it
kubectl -n argocd get applicationset infrastructure-appset -o yaml | \
  yq eval '.spec.generators[0].git.files'

# Check ArgoCD logs for errors
kubectl -n argocd logs -l app.kubernetes.io/name=argocd-applicationset-controller
```

**Sync failures:**
```bash
# Check app conditions
kubectl -n argocd get app <app-name> -o yaml | yq eval '.status.conditions'

# Force sync retry
kubectl -n argocd app sync <app-name>
```

### Storage Issues

**PVC stuck in Pending:**
```bash
# Check storage class
kubectl get sc

# Check PVC events
kubectl describe pvc <pvc-name>

# Verify Ceph is healthy
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

**Ceph OSD crashes:**
```bash
# Check OSD logs
kubectl -n rook-ceph logs -l app=rook-ceph-osd

# Verify disk is available
kubectl -n rook-ceph get cephcluster -o yaml | yq eval '.status.storage'
```

### Certificate Issues

**Certificate not issued:**
```bash
# Check certificate status
kubectl describe certificate -n <namespace> <cert-name>

# Check cert-manager logs
kubectl -n cert-manager logs -l app.kubernetes.io/name=cert-manager

# Verify ClusterIssuer is ready
kubectl get clusterissuers

# Check DNS challenge
kubectl -n cert-manager get challenges
kubectl -n cert-manager describe challenge <challenge-name>
```

### Secret Issues

**ExternalSecret not syncing:**
```bash
# Check ExternalSecret status
kubectl -n <namespace> get externalsecret <name> -o yaml | \
  yq eval '.status'

# Verify SecretStore connection
kubectl get clustersecretstore <store-name> -o yaml | \
  yq eval '.status.conditions'

# Check external-secrets logs
kubectl -n external-secrets logs -l app.kubernetes.io/name=external-secrets
```

### Scan Issues

**Trivy scan jobs failing:**
```bash
# Check scan job logs
kubectl -n trivy-system logs job/scan-vulnerabilityreport-<id>

# Check for rate limiting
kubectl -n trivy-system get events | grep -i "rate\|backoff"

# Verify Trivy server is running
kubectl -n trivy-system get pods | grep trivy-server

# Reduce concurrent scans in values.yaml:
# operator.scanJobsConcurrentLimit: 3
```

## CI/CD Reports (GitHub Actions)

### Where SARIF results appear

Trivy SARIF uploads are visible in GitHub:

- Repository 3 **Security** 3 **Code scanning**

### Daily container scan report

The scheduled container scan runs daily and reports results in:

1) **Actions run summary** (Affected images table)
2) **Artifacts** (downloadable Trivy table/SARIF files)
3) **Security 3 Code scanning** (SARIF)

### Enable optional Conftest policy checks

Conftest is disabled by default. To enable it:

- GitHub 3 Settings 3 Secrets and variables 3 Actions 3 Variables
- Add variable: `ENABLE_CONFTEST=true`

## Performance Tuning

### Reduce Resource Usage

For single-node development environments:

**Rook-Ceph:**
- Set `mon.count: 1` (default is 3)
- Set `mgr.count: 1` (default is 2)
- Set `storage.replicated.size: 1` (no redundancy)

**Istio:**
- Set `pilot.replicaCount: 1`
- Set `gateway.replicas: 1`

**ArgoCD:**
- Use single replica for all components
- Reduce resource requests

### Speed Up Deployments

1. **Pre-pull images:**
```bash
# Pull common images to avoid wait times
kubectl create job pre-pull --image=<image> -- /bin/true
```

2. **Increase ArgoCD sync concurrency:**
Edit `argocd/appsets/applicationsets.yaml`:
```yaml
spec:
  template:
    spec:
      syncPolicy:
        retry:
          limit: 5
          backoff:
            duration: 5s  # Faster initial retry
```

3. **Disable unnecessary scanners (Trivy):**
```yaml
operator:
  vulnerabilityScannerEnabled: true
  configAuditScannerEnabled: false  # Disable if not needed
  rbacAssessmentScannerEnabled: false
  exposedSecretScannerEnabled: false
```

## Best Practices

1. **Always use resource limits** to prevent runaway pods
2. **Set sync waves** for proper dependency ordering
3. **Use `createNamespace: true`** to avoid manual namespace creation
4. **Pin chart versions** for reproducible deployments
5. **Enable selfHeal** for automatic drift correction
6. **Use `prune: false`** initially to avoid accidental deletion
7. **Monitor scan reports** weekly for new CVEs
8. **Backup OpenBao** data regularly (stored in Ceph)
9. **Test PVC creation** before deploying stateful apps
10. **Check certificate expiry** 30 days before expiration

## Research Notes

This infrastructure supports thesis research on:
- **SSDF Compliance**: PW (Prepare the Organization), PO (Protect Software), PS (Produce Well-Secured Software), RV (Respond to Vulnerabilities)
- **SLSA Levels**: Source integrity, build integrity, provenance, dependencies
- **Supply Chain Security**: Vulnerability scanning, SBOM generation, signed artifacts
