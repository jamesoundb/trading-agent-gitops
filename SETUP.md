# Next Steps for Trading Agent GitOps Deployment

## 1. Create GitHub Repository
Go to https://github.com/new and create:
- **Repository name**: `trading-agent-gitops`
- **Visibility**: Public (ArgoCD needs to read it)
- **Initialize**: Without README (we already have one)
- **Add .gitignore**: None (we already have it)

## 2. Add Remote and Push

After creating the repo, run:

```bash
cd /home/james/trading-agent-gitops

# Add remote (replace YOUR_USERNAME with your GitHub username)
git remote add origin https://github.com/YOUR_USERNAME/trading-agent-gitops.git

# Push to main
git branch -M main
git push -u origin main
```

## 3. Deploy ArgoCD to Control Plane

```bash
# Set kubeconfig to control plane VIP
export KUBECONFIG=~/.kube/config

# Verify connection
kubectl cluster-info

# Create argocd namespace
kubectl create namespace argocd

# Deploy ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl rollout status deployment/argocd-server -n argocd
```

## 4. Create Kubernetes Secrets

**IMPORTANT**: Secrets are NOT in Git. Create them manually:

```bash
# Create the secret in the trading namespace (created by ArgoCD app)
kubectl create namespace trading

kubectl create secret generic trading-credentials \
  --from-literal=SCHWAB_CLIENT_ID="your-client-id" \
  --from-literal=SCHWAB_CLIENT_SECRET="your-client-secret" \
  --from-literal=SCHWAB_MCP_DISCORD_TOKEN="your-bot-token" \
  --from-literal=SCHWAB_MCP_DISCORD_CHANNEL_ID="1467642760105431196" \
  --from-literal=SCHWAB_MCP_DISCORD_APPROVERS="725433829288050698" \
  --from-literal=DISCORD_WEBHOOK_URL="your-webhook-url" \
  --from-literal=DRY_RUN="true" \
  -n trading
```

## 5. Create ArgoCD Application

First, update `argocd-app.yaml` with your GitHub username:

```bash
cd /home/james/trading-agent-gitops

# Edit argocd-app.yaml and change USERNAME to your GitHub username
# repoURL should be: https://github.com/YOUR_USERNAME/trading-agent-gitops.git

# Or do it with sed:
sed -i 's|USERNAME|your-github-username|g' argocd-app.yaml

# Apply the ArgoCD Application
kubectl apply -f argocd-app.yaml
```

## 6. Monitor Deployment

```bash
# Check ArgoCD app status
kubectl get applications -n argocd

# Watch deployment
kubectl logs -f -n trading deployment/trading-agent

# Check pod status
kubectl get pods -n trading -w

# Verify secret loaded
kubectl get secret -n trading trading-credentials
```

## 7. Verify Dry-Run Mode

```bash
# Confirm DRY_RUN is true
kubectl get secret -n trading trading-credentials -o jsonpath='{.data.DRY_RUN}' | base64 -d

# Check trading journal exists
kubectl exec -it -n trading $(kubectl get pod -n trading -l app=trading-agent -o jsonpath='{.items[0].metadata.name}') -- ls -la /app/journal/
```

## Expected Timeline

1. **GitHub repo creation**: 1 minute
2. **Push code**: 1 minute
3. **ArgoCD deploy**: 3-5 minutes
4. **Secrets creation**: 1 minute
5. **Application sync**: 2-3 minutes
6. **Pods starting**: 1-2 minutes

**Total**: ~10-15 minutes from push to operational trading agent

## Verification Checklist

- [ ] GitHub repo created and code pushed
- [ ] ArgoCD deployed to control plane
- [ ] Kubernetes secret created with credentials
- [ ] ArgoCD Application synced successfully
- [ ] Trading agent pod running in trading namespace
- [ ] DRY_RUN=true confirmed via secret
- [ ] trading_journal.csv receiving updates
- [ ] Discord receiving analysis posts

## If Something Goes Wrong

```bash
# Check ArgoCD app sync status and errors
kubectl describe application trading-agent -n argocd

# Check pod events
kubectl describe pod -n trading -l app=trading-agent

# View logs
kubectl logs -n trading -l app=trading-agent --previous

# Check secret exists
kubectl get secret -n trading trading-credentials -o yaml
```

## Next: Switch to Live Trading

Only after 4+ weeks of dry-run validation with >55% win rate:

```bash
# 1. Delete old secret
kubectl delete secret -n trading trading-credentials

# 2. Create new secret with DRY_RUN=false
kubectl create secret generic trading-credentials \
  --from-literal=SCHWAB_CLIENT_ID="..." \
  --from-literal=SCHWAB_CLIENT_SECRET="..." \
  --from-literal=SCHWAB_MCP_DISCORD_TOKEN="..." \
  --from-literal=SCHWAB_MCP_DISCORD_CHANNEL_ID="1467642760105431196" \
  --from-literal=SCHWAB_MCP_DISCORD_APPROVERS="725433829288050698" \
  --from-literal=DISCORD_WEBHOOK_URL="..." \
  --from-literal=DRY_RUN="false" \
  -n trading

# 3. Restart pods
kubectl rollout restart deployment/trading-agent -n trading

# 4. Monitor closely for first week
kubectl logs -f -n trading deployment/trading-agent
```

Good luck! 🚀
