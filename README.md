# Trading Agent GitOps

Kubernetes manifests for deploying the trading agent using ArgoCD GitOps workflow.

## Repository Structure

```
trading-agent-gitops/
├── manifests/          # Base manifests (Kustomize)
├── overlays/
│   ├── dev/           # DRY_RUN=true for paper trading
│   └── prod/          # DRY_RUN=false for live trading
├── argocd-app.yaml    # ArgoCD Application definition
└── README.md
```

## Quick Start

### 1. Create ArgoCD Application

```bash
kubectl apply -f argocd-app.yaml
```

### 2. Monitor Deployment

```bash
# Check ArgoCD sync status
kubectl get applications -n argocd

# View logs
kubectl logs -f -n trading deployment/trading-agent

# Check pod status
kubectl get pods -n trading
```

### 3. Verify Dry-Run Mode

```bash
# Should show DRY_RUN=true in pods
kubectl get secret -n trading trading-credentials -o jsonpath='{.data.DRY_RUN}' | base64 -d

# Check trading journal
kubectl exec -it -n trading $(kubectl get pod -n trading -l app=trading-agent -o jsonpath='{.items[0].metadata.name}') -- tail -20 /app/journal/trading_journal.csv
```

## Configuration

### Update Strategy Parameters

Edit `manifests/configmap.yaml`:
- `wheel_config.yaml` - Wheel strategy symbols, yields, DTE
- `bitcoin_momentum_config.yaml` - Bitcoin timeframes, SMA periods, RSI thresholds

Changes auto-sync from Git via ArgoCD.

### Update Credentials

**Important**: Credentials are stored in Kubernetes Secret, not in Git.

```bash
# Update secret (not in Git)
kubectl delete secret -n trading trading-credentials
kubectl create secret generic trading-credentials \
  --from-literal=SCHWAB_CLIENT_ID="your-key" \
  --from-literal=SCHWAB_CLIENT_SECRET="your-secret" \
  --from-literal=DISCORD_WEBHOOK_URL="your-webhook" \
  --from-literal=DRY_RUN="true" \
  -n trading

# Restart pods to pick up new secrets
kubectl rollout restart deployment/trading-agent -n trading
```

### Switch from DRY-RUN to LIVE

```bash
# 1. Update secret
kubectl delete secret -n trading trading-credentials
kubectl create secret generic trading-credentials \
  ... (same values) \
  --from-literal=DRY_RUN="false" \
  -n trading

# 2. Restart agent
kubectl rollout restart deployment/trading-agent -n trading

# 3. Monitor closely
kubectl logs -f -n trading deployment/trading-agent
```

## Monitoring

### Check Logs

```bash
# Trading agent logs
kubectl logs -f -n trading deployment/trading-agent

# Schwab MCP server logs
kubectl logs -f -n trading deployment/schwab-mcp-server
```

### Check Resources

```bash
# See resource usage
kubectl top pod -n trading

# Check PVC usage
kubectl describe pvc -n trading trading-journal-pvc
```

### Access Trading Journal

```bash
# Copy from PVC
kubectl cp trading/$(kubectl get pod -n trading -l app=trading-agent -o jsonpath='{.items[0].metadata.name}'):/app/journal/trading_journal.csv ./trading_journal.csv
```

## Troubleshooting

### Pods Not Starting

```bash
# Check events
kubectl describe pod -n trading -l app=trading-agent

# Check previous logs
kubectl logs -n trading -l app=trading-agent --previous
```

### Secret Not Loading

```bash
# Verify secret exists
kubectl get secret -n trading trading-credentials -o yaml

# Check if pod can access it
kubectl exec -it -n trading $(kubectl get pod -n trading -l app=trading-agent -o jsonpath='{.items[0].metadata.name}') -- env | grep DISCORD
```

### PVC Issues

```bash
# Check PVC status
kubectl describe pvc -n trading trading-journal-pvc

# Check available storage
kubectl get pv
```

## GitOps Workflow

### Making Changes

1. **Update manifests in Git**
   ```bash
   git checkout -b feature/new-strategy
   # Edit manifests/configmap.yaml
   git commit -am "Add new Bitcoin strategy parameters"
   git push origin feature/new-strategy
   ```

2. **Create Pull Request** (for approval)

3. **Merge to main**
   ```bash
   git checkout main
   git merge feature/new-strategy
   git push origin main
   ```

4. **ArgoCD auto-syncs** (within 3 minutes or manually trigger)
   ```bash
   argocd app sync trading-agent
   ```

5. **Monitor deployment**
   ```bash
   kubectl rollout status deployment/trading-agent -n trading
   ```

### Rollback

```bash
# Revert Git commit
git revert HEAD
git push

# ArgoCD syncs back to previous version
argocd app sync trading-agent
```

## Deployment Locations

### Control Plane (ARM64 - Raspberry Pi)
- Schwab MCP server runs here
- Lower latency for API calls

### Worker Nodes (AMD64 - Proxmox VMs)
- Trading agent runs here
- Better resource isolation
- Can scale independently

## Security Notes

- ⚠️ **Secrets NOT in Git** - Created manually in cluster
- ✅ ConfigMaps (non-sensitive data) in Git
- ✅ Use Sealed Secrets for production (future enhancement)
- ✅ RBAC configured per service

## Next Steps

1. Create public GitHub repo named `trading-agent-gitops`
2. Push this repo to remote
3. Configure ArgoCD Application to point to your repo
4. Monitor first sync via `kubectl get applications -n argocd`

See `K8S_DEPLOYMENT_STRATEGY.md` in the Investing repo for detailed instructions.
