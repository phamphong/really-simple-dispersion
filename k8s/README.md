# Kubernetes Configuration for ECR Credentials Update

This directory contains Kubernetes manifests for automating ECR (Amazon Elastic Container Registry) credentials updates across multiple namespaces.

## Files Overview

- **`ecr-creds-update-cronjob.yaml`** - Main CronJob that updates ECR credentials every 10 minutes
- **`rbac.yaml`** - ServiceAccount, ClusterRole, and ClusterRoleBinding for permissions
- **`configmaps-secrets-example.yaml`** - Example ConfigMaps and Secrets (needs customization)
- **`ECR-CRONJOB-EXPLANATION.md`** - Detailed explanation in Vietnamese and English

## Quick Start

1. **Customize the configuration**:
   ```bash
   cp configmaps-secrets-example.yaml configmaps-secrets.yaml
   # Edit configmaps-secrets.yaml with your actual AWS credentials and ECR details
   ```

2. **Apply the configurations**:
   ```bash
   # Apply RBAC first
   kubectl apply -f rbac.yaml
   
   # Apply your customized secrets and configmaps
   kubectl apply -f configmaps-secrets.yaml
   
   # Apply the CronJob
   kubectl apply -f ecr-creds-update-cronjob.yaml
   ```

3. **Verify the deployment**:
   ```bash
   kubectl get cronjob ecr-creds-update -n kube-system
   kubectl describe cronjob ecr-creds-update -n kube-system
   ```

## Monitoring

```bash
# Check recent jobs
kubectl get jobs -n kube-system | grep ecr-creds-update

# View logs
kubectl logs -l job-name=ecr-creds-update -n kube-system --tail=50

# Check updated secrets
kubectl get secret ecr-registry-secret -n default -o yaml
```

## Security Notes

⚠️ **Important**: 
- Never commit actual AWS credentials to version control
- Use Kubernetes secrets management best practices
- Consider using IAM roles for service accounts (IRSA) in production
- Regularly rotate AWS credentials

## Customization Required

Before deploying, you must update:
1. AWS credentials in the secret
2. ECR registry URL
3. Target namespaces list
4. Email address for Docker registry

See `ECR-CRONJOB-EXPLANATION.md` for detailed configuration instructions.