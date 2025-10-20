# Money-Moving GitOps Configuration

GitOps repository for managing Money-Moving deployments across multiple environments using ArgoCD and Kustomize.

## 📁 Repository Structure

```
money-moving-config/
├── argocd-apps/              # ArgoCD Application definitions
│   ├── dev.yaml              # Dev environment Application
│   ├── qa.yaml               # QA environment Application
│   ├── staging.yaml          # Staging environment Application
│   └── production-project.yaml # AppProject for staging
├── base/                     # Base Kubernetes manifests
│   ├── kustomization.yaml
│   ├── shared-configmap.yaml
│   ├── ghcr-secret.yaml
│   ├── ledger-deployment.yaml
│   ├── ledger-service.yaml
│   ├── ledger-backoffice-deployment.yaml
│   ├── ledger-backoffice-service.yaml
│   └── ledger-backoffice-ingress.yaml  # HTTPS with Let's Encrypt
└── environments/             # Environment-specific overlays
    ├── dev/
    │   ├── kustomization.yaml
    │   └── shared-external-secret.yaml
    ├── qa/
    │   ├── kustomization.yaml
    │   └── shared-secret.yaml
    └── staging/
        └── kustomization.yaml
```

## 🏗️ Architecture

### Applications

- **ledger**: Ledger v2 service (TigerBeetle-based accounting)
  - gRPC API (50051)
  - HTTP REST API (8080)

- **ledger-backoffice**: Web UI for ledger management
  - Next.js application (3001)
  - Public access: https://ledger-backoffice.londonbridge.dev
  - Internal-only (VPN required)

All applications:
- Deploy to namespace: `money-moving`
- Use shared ConfigMap and ExternalSecret
- Pull images from GHCR: `ghcr.io/london-bridge/money-moving/`

### Environments

| Environment | Replicas | Resources | Sync Policy | Project |
|-------------|----------|-----------|-------------|---------|
| **Dev** | 1 | 50m CPU / 200Mi RAM | Automated | default |
| **QA** | 1 | 100m CPU / 300Mi RAM | Automated | default |
| **Staging** | 2 | 200m CPU / 512Mi RAM | Manual | money-moving-production |

## 🚀 Deployment Flow

```
main branch → DEV (auto)
    ↓
   QA tag → QA (auto after approval)
    ↓
  STG tag → STG (PR → approval → merge → manual sync)
```

### 1. Development Environment

**Trigger**: Push to `main` branch in money-moving repo

**Process**:
1. CI/CD builds Docker images with tag `main-{sha}`
2. GitHub Actions updates `environments/dev/kustomization.yaml`
3. Commits directly to main branch
4. ArgoCD auto-syncs (prune + selfHeal enabled)

**Deployment Names**: `dev-ledger`

### 2. QA Environment

**Trigger**: Manual promotion workflow or `qa-*` tag

**Process**:
1. Validate images exist in GHCR
2. Update `environments/qa/kustomization.yaml` with `qa-{sha}` tag
3. Direct commit to main
4. ArgoCD auto-syncs

**Deployment Names**: `qa-ledger`

### 3. Staging Environment

**Trigger**: Manual promotion workflow or `stg-*` tag

**Process**:
1. Create Pull Request updating `environments/staging/kustomization.yaml`
2. Require approvals (QA Team + Tech Lead)
3. After merge, manually trigger ArgoCD sync
4. 2 replicas for HA
5. Production-mirror configuration

**Deployment Names**: `stg-ledger`

## 📦 How to Update

### Update Dev (Automatic)

Automated via CI/CD workflow after successful build on `main` branch.

### Update QA (Manual)

```bash
# In money-moving repository
gh workflow run promote-qa.yml -f COMMIT_SHA=abc123
```

### Update Staging (Manual via PR)

```bash
# In money-moving repository
gh workflow run promote-staging.yml -f COMMIT_SHA=abc123
```

Or manually:

```bash
# Clone this repo
git clone https://github.com/london-bridge/money-moving-config
cd money-moving-config

# Create branch
git checkout -b promote-staging-abc123

# Update image tags in environments/staging/kustomization.yaml
vi environments/staging/kustomization.yaml

# Commit and create PR
git add environments/staging/kustomization.yaml
git commit -m "feat(staging): promote ledger to abc123"
git push origin promote-staging-abc123
gh pr create --title "Promote Staging to abc123" --body "..."
```

## 🔧 Local Testing

### Validate Kustomization

```bash
# Dev
kustomize build environments/dev

# QA
kustomize build environments/qa

# Staging
kustomize build environments/staging
```

### Apply Manually (for testing)

```bash
# Dev
kustomize build environments/dev | kubectl apply -f -

# Check deployment
kubectl get pods -n money-moving
kubectl get deployments -n money-moving
```

## 🔐 Secrets Management

Secrets are managed via **External Secrets Operator**:

- **ClusterSecretStore**: `aws-secret-store`
- **AWS Secrets Manager** paths:
  - Dev: `/money-moving/dev/manual-secrets`
  - QA: `/money-moving/qa/manual-secrets`
  - Staging: `/money-moving/staging/manual-secrets`

To update secrets:

```bash
# Update in AWS Secrets Manager
aws secretsmanager update-secret \
  --secret-id /money-moving/dev/manual-secrets \
  --secret-string '{
    "TIGERBEETLE_CLUSTER_ID": "0",
    "TIGERBEETLE_ADDRESSES": "tigerbeetle-0:3000,tigerbeetle-1:3001,tigerbeetle-2:3002",
    "DATABASE_URL": "postgresql://..."
  }'

# ExternalSecret will auto-refresh every 15s
```

## 📊 ArgoCD Setup

### Install Applications

```bash
# Apply ArgoCD Project (staging only)
kubectl apply -f argocd-apps/production-project.yaml -n argocd

# Apply Applications (in respective ArgoCD instances)
# Dev ArgoCD
kubectl apply -f argocd-apps/dev.yaml -n argocd

# QA ArgoCD
kubectl apply -f argocd-apps/qa.yaml -n argocd

# Staging ArgoCD
kubectl apply -f argocd-apps/staging.yaml -n argocd
```

### View in ArgoCD UI

```bash
# Dev
argocd app get money-moving-dev

# QA
argocd app get money-moving-qa

# Staging
argocd app get money-moving-staging
```

### Manual Sync

```bash
# Staging (manual sync required)
argocd app sync money-moving-staging
```

## 🐛 Troubleshooting

### Application Out of Sync

```bash
# Check diff
argocd app diff money-moving-dev

# Force sync
argocd app sync money-moving-dev --force

# Refresh
argocd app get money-moving-dev --refresh
```

### Pod Failing to Start

```bash
# Check pod logs
kubectl logs -n money-moving dev-ledger-xxx

# Check events
kubectl describe pod -n money-moving dev-ledger-xxx

# Check secrets
kubectl get externalsecret -n money-moving
kubectl get secret money-moving-shared-secret -n money-moving
```

### Image Pull Errors

```bash
# Check ghcr-secret
kubectl get secret ghcr-secret -n money-moving -o yaml

# Recreate secret if needed
kubectl delete secret ghcr-secret -n money-moving
kubectl apply -f base/ghcr-secret.yaml
```

### Rollback

```bash
# Option 1: Git revert
git revert HEAD
git push

# Option 2: Kubectl rollback
kubectl rollout undo deployment/dev-ledger -n money-moving

# Option 3: Update image tag in kustomization.yaml to previous version
```

## 📚 Services

### Ledger v2

**Technology Stack**:
- Go
- TigerBeetle (financial transactions database)
- Clean Architecture + DDD
- HTTP REST API (8080)
- gRPC API (50051)

**Endpoints**:
- `GET /health` - Health check
- `GET /ready` - Readiness check
- `POST /api/v1/accounts` - Create account
- `POST /api/v1/transfers` - Create transfer
- `GET /api/v1/accounts/:id` - Get account details

**Documentation**: See [money-moving repository](https://github.com/london-bridge/money-moving/tree/main/apps/ledger-v2/_docs)

### Ledger Backoffice

**Technology Stack**:
- Next.js 15 (React)
- TypeScript
- Tailwind CSS
- gRPC client (connects to Ledger v2)

**Access**:
- **URL**: https://ledger-backoffice.londonbridge.dev
- **Requirements**: VPN connection required (DEV environment)
- **Authentication**: Internal access only (IP whitelist: VPN + VPC)

**Features**:
- Account management UI
- Transfer history visualization
- Real-time balance tracking
- Transaction search and filters

**No Configuration Needed**:
- ✅ DNS configured automatically via Route53
- ✅ HTTPS with Let's Encrypt certificate
- ✅ No port-forward required
- ✅ Zero setup for developers

**Troubleshooting**:

If you cannot access the backoffice:

1. **Verify VPN connection**:
   ```bash
   # You should be on 10.8.0.0/24 network
   ip addr show tun0
   ```

2. **Check DNS resolution**:
   ```bash
   nslookup ledger-backoffice.londonbridge.dev
   # Should resolve to internal ELB
   ```

3. **Clear DNS cache** (if DNS not resolving):
   ```bash
   sudo systemctl restart systemd-resolved
   ```

4. **Test direct access**:
   ```bash
   curl -I https://ledger-backoffice.londonbridge.dev
   # Should return HTTP/2 200
   ```

5. **Check certificate**:
   ```bash
   kubectl get certificate ledger-backoffice-tls-secret -n money-moving
   # Should show READY = True
   ```

**Architecture**:
```
Browser (VPN)
    ↓ HTTPS
Internal Load Balancer (internal-nginx)
    ↓ HTTP
ledger-backoffice Service (ClusterIP:3001)
    ↓ gRPC
ledger Service (ClusterIP:50051)
    ↓
TigerBeetle Cluster
```

## 🤝 Contributing

1. Create feature branch
2. Update manifests
3. Test locally with `kustomize build`
4. Create Pull Request
5. Wait for approvals (staging only)
6. Merge to main

## 📝 Notes

- **Environment isolation** - Each environment has its own configuration
- **Shared resources** - ConfigMap and ExternalSecret are shared across services
- **Atomic deployments** - Changes apply to all services simultaneously
- **Environment progression** - Dev → QA → Staging flow
- **Based on**: [orchestration-go-config](https://github.com/london-bridge/orchestration-go-config) patterns

## 📚 Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [External Secrets Operator](https://external-secrets.io/)
- [Main Repository](https://github.com/london-bridge/money-moving)
