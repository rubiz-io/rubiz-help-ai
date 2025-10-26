# Rubiz Help AI - Deployment Repository

AI-powered help system for Rubiz VPN using the shared `flash-help-ai` Docker image.

## Architecture

This is a **deployment-only repository**. It does not contain source code or build Docker images.

- **Source Code**: Maintained in `flash-help-ai` repository
- **Docker Image**: Uses `gcr.io/flashvpn-253908/flash-help-ai:latest`
- **This Repo Contains**: Kubernetes manifests and deployment workflows
- **Namespace**: `rubiz`
- **ChromaDB Collection**: `rubiz-help`

## Directory Structure

```
rubiz-help-ai/
├── .github/
│   └── workflows/
│       └── deploy.yml          # Deployment pipeline
├── k8s/
│   ├── uat/
│   │   ├── deployment.yaml     # UAT deployment config
│   │   └── ingress.yaml        # UAT Traefik IngressRoute
│   └── prod/
│       ├── deployment.yaml     # Production deployment config
│       └── ingress.yaml        # Production Traefik IngressRoute
└── README.md
```

## Deployment Workflow

### Branch Strategy

| Branch | Environment | Auto-Deploy | Approval Required |
|--------|-------------|-------------|-------------------|
| `main` | Production | Yes | ✅ Required |
| `release/**` | UAT | Yes | ❌ No |
| `hotfix/**` | UAT | Yes | ❌ No |

### Deployment Process

**1. Deploy to UAT (Testing)**
```bash
# Create release branch
git checkout -b release/v1.0.0

# Push to trigger UAT deployment
git push origin release/v1.0.0

# Automatically deploys to uat-home.rubizvpn.com
```

**2. Deploy to Production**
```bash
# Merge to main
git checkout main
git merge release/v1.0.0

# Push to trigger production deployment (requires manual approval)
git push origin main

# After approval, deploys to home.rubizvpn.com
```

**3. Hotfix Deployment**
```bash
# Create hotfix branch
git checkout -b hotfix/critical-bug

# Push to test in UAT
git push origin hotfix/critical-bug

# After verification, merge to main for production
git checkout main
git merge hotfix/critical-bug
git push origin main
```

## Required GitHub Secrets

### Rubiz-Specific Secrets

| Secret | Description | Required |
|--------|-------------|----------|
| `RUBIZ_HELP_DOMAIN` | Help domain (e.g., guides.rubizvpn.com) | ✅ Yes |
| `RUBIZ_NOTION_TOKEN` | Notion API token | ✅ Yes |
| `RUBIZ_NOTION_DATABASE_ID` | Notion database ID | ✅ Yes |
| `RUBIZ_CHATWOOT_ACCOUNT_ID` | Chatwoot account ID | Optional |
| `RUBIZ_CHATWOOT_INBOX_ID` | Chatwoot inbox ID | Optional |
| `RUBIZ_CHATWOOT_API_ACCESS_TOKEN` | Chatwoot API token | Optional |
| `RUBIZ_CHATWOOT_API_BASE_URL` | Chatwoot API base URL | Optional |
| `RUBIZ_CHATWOOT_WEBHOOK_SECRET` | Chatwoot webhook secret | Optional |
| `RUBIZ_NOTION_WEBHOOK_SECRET` | Notion webhook secret | Optional |

### Shared Secrets (Used by All Brands)

| Secret | Description | Required |
|--------|-------------|----------|
| `OPENAI_API_KEY` | OpenAI API key | ✅ Yes |
| `DIGITALOCEAN_ACCESS_TOKEN` | DigitalOcean access token | ✅ Yes |
| `DO_CLUSTER_NAME_UAT` | UAT K8s cluster name | ✅ Yes |
| `DO_CLUSTER_NAME_PROD` | Production K8s cluster name | ✅ Yes |
| `SLACK_WEBHOOK` | Slack webhook for notifications | Optional |

### Optional Configuration Secrets

| Secret | Description | Default |
|--------|-------------|---------|
| `LOG_LEVEL` | Logging level | `info` |
| `MAX_RETRIES` | Max retry attempts | `3` |
| `TIMEOUT_MS` | Request timeout | `30000` |
| `RATE_LIMIT_MAX` | Rate limit max requests | `100` |
| `RATE_LIMIT_WINDOW_MS` | Rate limit window | `60000` |

## Environment Configuration

The application is configured via Kubernetes secrets created during deployment:

| Variable | Value | Description |
|----------|-------|-------------|
| `CHROMA_COLLECTION` | `rubiz-help` | Rubiz ChromaDB collection |
| `HELP_DOMAIN` | `guides.rubizvpn.com` | Rubiz help domain |
| `REDIS_HOST` | `redis.default.svc.cluster.local` | Shared Redis |
| `CHROMA_HOST` | `chromadb.default.svc.cluster.local` | Shared ChromaDB |

## Monitoring & Troubleshooting

### Check Deployment Status

```bash
# Get kubeconfig for cluster
doctl kubernetes cluster kubeconfig save <cluster-name>

# Switch to Rubiz namespace
kubectl config set-context --current --namespace=rubiz

# Check deployment status
kubectl get deployments
kubectl get pods
kubectl get ingressroute

# Check logs
kubectl logs -l app=rubiz-help-ai --tail=100 -f

# Check pod details
kubectl describe pod <pod-name>
```

### Common Issues

**Issue: Workflow not triggering**
- Verify branch name matches: `main`, `release/*`, or `hotfix/*`
- Check GitHub Actions tab for workflow runs
- Ensure secrets are configured in GitHub repository settings

**Issue: Deployment fails with image pull error**
- Verify `gcr-flash` secret exists in `rubiz` namespace
- Ensure GCP service account has permissions to pull from GCR
- Check if flash-help-ai image exists and is tagged as `latest`

**Issue: Pod crashes or won't start**
- Check logs: `kubectl logs -l app=rubiz-help-ai`
- Verify all required secrets are set correctly
- Check Notion/OpenAI/Chatwoot credentials are valid

**Issue: IngressRoute not working**
- Verify Traefik is running in cluster
- Check middleware exists: `kubectl get middleware -n rubiz`
- Verify DNS points to cluster load balancer

### Manual Rollback

```bash
# Rollback to previous version
kubectl rollout undo deployment/rubiz-help-ai -n rubiz

# Check rollout status
kubectl rollout status deployment/rubiz-help-ai -n rubiz

# View rollout history
kubectl rollout history deployment/rubiz-help-ai -n rubiz
```

## Updating the Docker Image

To update the Docker image used by Rubiz Help AI:

1. Update and deploy the `flash-help-ai` repository
2. Flash Help AI builds and pushes new image to GCR
3. Deploy this repository to pull and use the new image:
   ```bash
   # Force pod restart to pull latest image
   kubectl rollout restart deployment/rubiz-help-ai -n rubiz
   ```

## Infrastructure

### Kubernetes Resources

**UAT Environment:**
- Replicas: 1
- CPU: 200m (request) - 500m (limit)
- Memory: 256Mi (request) - 512Mi (limit)
- Auto-scaling: 1-6 replicas

**Production Environment:**
- Replicas: 3
- CPU: 300m (request) - 800m (limit)
- Memory: 512Mi (request) - 1Gi (limit)
- Auto-scaling: 3-10 replicas
- Pod Disruption Budget: Min 2 available

### Shared Services

All brands share these services in the `default` namespace:
- **ChromaDB**: Vector database for semantic search
- **Redis**: Caching layer

## Endpoints

**UAT:**
- Health: `https://uat-home.rubizvpn.com/help-ai/health`
- API: `https://uat-home.rubizvpn.com/help-ai/*`

**Production:**
- Health: `https://home.rubizvpn.com/help-ai/health`
- API: `https://home.rubizvpn.com/help-ai/*`

## Support

For issues or questions:
1. Check deployment logs in GitHub Actions
2. Review Kubernetes pod logs
3. Contact DevOps team
4. Create issue in this repository

---

**Related Repositories:**
- Source Code: `flash-help-ai`
- JetStream Deployment: `jss-help-ai`
- Aurumn Deployment: `aurumn-help-ai`
