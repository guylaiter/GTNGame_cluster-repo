# FinalProject Cluster Configuration

GitOps configuration for deploying the "Guess The Number" application to Kubernetes using Helm and ArgoCD.

## Overview

This repository contains Kubernetes manifests and ArgoCD configuration for the complete application stack:
- **Backend:** Flask REST API
- **Database:** PostgreSQL with persistent storage
- **Secrets:** AWS Secrets Manager integration via CSI driver
- **GitOps:** ArgoCD for automated deployments


## Prerequisites

Before using this repository, you must have:

### 1. Infrastructure Setup (from infrastructure-repo)
- ✅ EKS cluster running in AWS
- ✅ ArgoCD installed and configured
- ✅ AWS Secrets Store CSI Driver installed
- ✅ IAM permissions for Secrets Manager access

### 2. AWS Secrets Manager Secret
Create a secret named `guess-the-number/database` with:
```json
{
  "username": "postgres",
  "password": "YOUR_SECURE_PASSWORD",
  "database": "guessdb",
  "host": "guess-the-number-postgres-service",
  "port": "5432"
}
```

### 3. Container Image
- Docker image built and pushed to ECR (handled by app-repo CI/CD)
- Image repository: `900631658007.dkr.ecr.ap-south-1.amazonaws.com/final-project-app-repo`

## Quick Setup

### Step 1: Clone This Repository
```bash
git clone https://github.com/guylaiter/FinalProject_cluster-repo.git
cd FinalProject_cluster-repo
```

### Step 2: Update Image Tag (if needed)
Edit `helm/values.yaml` to set the desired image version:
```yaml
backend:
  image:
    tag: v1.0.0  # Update this to your image version
```

### Step 3: Deploy ArgoCD Application
```bash
kubectl apply -f argocd/application.yaml
```

### Step 4: Verify Deployment
```bash
# Watch ArgoCD sync the application
kubectl get application -n argocd guess-the-number

# Watch pods come up
kubectl get pods -w

# Check all resources
kubectl get all
```

### Step 5: Get LoadBalancer URL
```bash
# Get the external URL (takes 2-3 minutes for AWS to provision)
kubectl get svc guess-the-number-backend-service

# Or get URL directly:
kubectl get svc guess-the-number-backend-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Step 6: Access Your Application
```bash
# Open in browser
http://<LOAD_BALANCER_URL>
```

## GitOps Workflow

This repository follows GitOps principles:

### Making Changes
1. Edit files in this repository (e.g., `helm/values.yaml`)
2. Commit and push to GitHub
3. ArgoCD automatically detects changes within ~3 minutes
4. ArgoCD deploys the changes to your cluster

### Example: Deploy New Version
```bash
# Update image tag in values.yaml
sed -i 's/tag: .*/tag: v1.0.5/' helm/values.yaml

# Commit and push
git add helm/values.yaml
git commit -m "Deploy v1.0.5"
git push origin main

# ArgoCD automatically deploys! ✅
```

### Auto-Sync Features
- **Prune:** Automatically deletes resources removed from Git
- **Self-Heal:** Reverts manual changes made directly to the cluster
- **Retry:** Automatically retries failed deployments with backoff

## Configuration

### Scaling the Application
Edit `helm/values.yaml`:
```yaml
backend:
  replicas: 3  # Increase for more capacity
```

### Updating the Secret Number
Edit `helm/values.yaml`:
```yaml
backend:
  config:
    secretNumber: "100"  # Change the game secret
```

### Adjusting Resources
Edit `helm/values.yaml`:
```yaml
backend:
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "512Mi"
      cpu: "500m"
```

### Database Storage
Edit `helm/values.yaml`:
```yaml
database:
  storage:
    size: 5Gi  # Increase for more storage
```

## Repository Structure

```
.
├── argocd/
│   └── application.yaml              # ArgoCD Application definition
│                                     # - Points to this repo
│                                     # - Enables auto-sync
│                                     # - Configures GitOps behavior
│
├── helm/
│   ├── Chart.yaml                    # Helm chart metadata
│   │                                 # - Chart name and version
│   │                                 # - App version tracking
│   │
│   ├── values.yaml                   # Configuration values
│   │                                 # - Image repository and tags
│   │                                 # - Replica counts
│   │                                 # - Resource limits
│   │                                 # - Database settings
│   │                                 # - All configurable parameters
│   │
│   └── templates/                    # Kubernetes manifests (Helm templates)
│       ├── configmap.yaml            # Non-sensitive configuration
│       │                             # - SECRET_NUMBER
│       │                             # - FLASK_ENV
│       │
│       ├── secret-provider.yaml      # AWS Secrets Manager integration
│       │                             # - SecretProviderClass for CSI driver
│       │                             # - Syncs AWS secrets to K8s
│       │
│       ├── postgres-pvc.yaml         # PostgreSQL storage
│       │                             # - PersistentVolumeClaim (AWS EBS)
│       │                             # - 1Gi by default
│       │
│       ├── postgres-statefulset.yaml # PostgreSQL database
│       │                             # - StatefulSet for stable identity
│       │                             # - Mounts persistent storage
│       │                             # - Reads credentials from secret
│       │
│       ├── postgres-service.yaml     # PostgreSQL networking
│       │                             # - ClusterIP service
│       │                             # - Internal access only
│       │
│       ├── deployment.yaml           # Backend Flask application
│       │                             # - 2 replicas by default
│       │                             # - Environment variables from ConfigMap/Secret
│       │                             # - Health checks (liveness/readiness)
│       │
│       └── service.yaml              # Backend LoadBalancer
│                                     # - Exposes app to internet
│                                     # - AWS ELB on port 80
│
└── README.md                         # This file
```

## Troubleshooting

### ArgoCD Application Not Syncing
```bash
# Check application status
kubectl describe application guess-the-number -n argocd

# Force sync
kubectl patch application guess-the-number -n argocd --type merge -p '{"operation":{"sync":{}}}'
```

### Pods Not Starting
```bash
# Check pod status
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Check events
kubectl get events --sort-by='.lastTimestamp'
```

### Database Connection Issues
```bash
# Check if secret was created
kubectl get secret guess-the-number-db-secret

# Verify secret contents
kubectl get secret guess-the-number-db-secret -o jsonpath='{.data.username}' | base64 -d

# Check PostgreSQL logs
kubectl logs guess-the-number-postgres-0
```

### LoadBalancer URL Not Appearing
```bash
# Check service status
kubectl describe svc guess-the-number-backend-service

# AWS ELB takes 2-3 minutes to provision
# Check AWS Console → EC2 → Load Balancers
```

## Monitoring

### View Application in ArgoCD UI
```bash
# Port-forward to ArgoCD (if not already running)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open browser
https://localhost:8080

# Login with credentials from infrastructure-repo setup
```

### Check Application Health
```bash
# Overall status
kubectl get application -n argocd

# Detailed view
kubectl describe application guess-the-number -n argocd
```

### Watch Deployment Progress
```bash
# Watch pods
kubectl get pods -w

# Watch all resources
kubectl get all -w
```

## CI/CD Integration

This repository is designed to work with the app-repo CI/CD pipeline:

1. **Developer pushes code** to app-repo
2. **GitHub Actions** builds Docker image → `v1.0.X`
3. **CI updates** `helm/values.yaml` in this repo with new tag
4. **ArgoCD detects change** and deploys automatically

*Note: CI/CD integration to update this repo is a future enhancement.*

## Security Notes

- ✅ Database credentials stored in AWS Secrets Manager (encrypted)
- ✅ IAM-based access control for secrets
- ✅ Secrets never committed to Git
- ✅ PostgreSQL not exposed to internet (ClusterIP only)
- ⚠️  LoadBalancer is public (consider adding authentication/authorization)

## Cost Estimate

Resources deployed by this configuration:

- **PostgreSQL Storage (EBS):** ~$0.10/GB/month (1GB = ~$0.10/month)
- **LoadBalancer (ELB):** ~$16/month + data transfer
- **Compute:** Covered by EKS nodes (from infrastructure-repo)

**Total additional cost: ~$16-20/month**

## Related Documentation

- **Infrastructure Setup:** See [FinalProject_infrastructure-repo](https://github.com/guylaiter/FinalProject_infrastructure-repo)
- **Application Code:** See [FinalProject_app-repo](https://github.com/guylaiter/FinalProject_app-repo)
- **Helm Documentation:** https://helm.sh/docs/
- **ArgoCD Documentation:** https://argo-cd.readthedocs.io/

## License

This project is part of a final course project.
