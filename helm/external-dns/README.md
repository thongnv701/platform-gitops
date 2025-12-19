# External-DNS Setup with Cloudflare

This directory contains the Helm values configuration for External-DNS, which automatically manages DNS records in Cloudflare based on Kubernetes Ingress resources.

## Prerequisites

- Cloudflare account with access to the `thongit.space` domain
- Kubernetes cluster with ArgoCD installed
- kubectl configured to access your cluster

## Setup Instructions

### Step 1: Create Cloudflare API Token

1. Log in to your Cloudflare dashboard: https://dash.cloudflare.com/
2. Go to **My Profile** → **API Tokens** (or visit https://dash.cloudflare.com/profile/api-tokens)
3. Click **Create Token**
4. Use the **Edit zone DNS** template, or create a custom token with the following permissions:
   - **Zone** → **Zone** → **Read**
   - **Zone** → **DNS** → **Edit**
5. Set **Zone Resources** to:
   - **Include** → **Specific zone** → `thongit.space`
6. Click **Continue to summary** and then **Create Token**
7. **Copy the API token immediately** (you won't be able to see it again)

### Step 2: Create Kubernetes Secret

Create a Kubernetes secret with the Cloudflare API token in the `external-dns` namespace:

```bash
# Create the namespace first (if it doesn't exist)
kubectl create namespace external-dns

# Create the secret with your Cloudflare API token
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token='YOUR_CLOUDFLARE_API_TOKEN_HERE' \
  --namespace=external-dns
```

**Important:** Replace `YOUR_CLOUDFLARE_API_TOKEN_HERE` with the actual API token you created in Step 1.

### Step 3: Verify Secret Creation

Verify the secret was created correctly:

```bash
kubectl get secret cloudflare-api-token -n external-dns
```

### Step 4: Deploy External-DNS via ArgoCD

1. Apply the ArgoCD Application:
   ```bash
   kubectl apply -f ../../apps/external-dns-app.yaml
   ```

2. Or if ArgoCD app-of-apps is configured, it should automatically pick up the new application.

3. Check the application status in ArgoCD UI or via CLI:
   ```bash
   kubectl get application external-dns -n argocd
   ```

4. Sync the application (if not automated):
   ```bash
   argocd app sync external-dns
   ```

### Step 5: Verify External-DNS Deployment

1. Check if External-DNS pod is running:
   ```bash
   kubectl get pods -n external-dns
   ```

2. Check External-DNS logs:
   ```bash
   kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns
   ```

3. Verify DNS records are being created in Cloudflare dashboard or via DNS lookup:
   ```bash
   dig shopacc.thongit.space
   dig api.thongit.space
   dig portfolio.thongit.space
   ```

## How It Works

External-DNS watches for Kubernetes Ingress resources and automatically:

1. **Creates DNS records** in Cloudflare when a new Ingress is created
2. **Updates DNS records** when Ingress hostnames change
3. **Deletes DNS records** when Ingress resources are removed

### Domain Filtering

External-DNS is configured to only manage DNS records for the `thongit.space` domain. Any Ingress resources with hosts in this domain will automatically get DNS records created.

### Ownership Tracking

External-DNS uses TXT records to track which DNS records it manages:
- TXT record format: `external-dns-<record-name>`
- This ensures External-DNS only manages records it created
- Prevents conflicts with manually created DNS records

## Configuration

### Current Configuration

- **Provider**: Cloudflare
- **Source**: Ingress resources
- **Domain Filter**: `thongit.space`
- **Policy**: `sync` (create, update, delete)
- **Registry**: TXT records for ownership tracking
- **Sync Interval**: 1 minute

### Updating Configuration

To modify External-DNS configuration:

1. Edit `helm/external-dns/values.yaml` or `apps/external-dns-app.yaml`
2. Commit and push changes to Git
3. ArgoCD will automatically sync the changes (if automated sync is enabled)

## Troubleshooting

### External-DNS Pod Not Starting

1. Check if the secret exists:
   ```bash
   kubectl get secret cloudflare-api-token -n external-dns
   ```

2. Verify the secret contains the correct key:
   ```bash
   kubectl get secret cloudflare-api-token -n external-dns -o jsonpath='{.data.api-token}' | base64 -d
   ```

3. Check pod logs for errors:
   ```bash
   kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns
   ```

### DNS Records Not Being Created

1. Verify Ingress resources have the correct hostnames:
   ```bash
   kubectl get ingress --all-namespaces -o wide
   ```

2. Check External-DNS logs for errors:
   ```bash
   kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns
   ```

3. Verify Cloudflare API token has correct permissions:
   - Zone:Read
   - DNS:Edit

4. Check if domain filter matches your domain:
   - Current filter: `thongit.space`
   - Ingress hosts must match this domain

### DNS Records Being Deleted

External-DNS uses a `sync` policy, which means it will delete DNS records if the corresponding Ingress resource is removed. This is expected behavior.

To prevent deletion of specific records, you can:
- Use a different TXT owner ID
- Manually create DNS records outside of External-DNS management
- Use `upsert-only` policy (not recommended, as it won't clean up orphaned records)

## Security Notes

- **API Token vs Global API Key**: This setup uses Cloudflare API Token, which is more secure than Global API Key as it has limited permissions and can be scoped to specific zones.

- **Secret Management**: The Cloudflare API token is stored as a Kubernetes Secret. Consider using:
  - External Secrets Operator for better secret management
  - Sealed Secrets for GitOps-friendly secret encryption
  - Cloud provider secret management services

- **RBAC**: External-DNS creates its own ServiceAccount with appropriate RBAC permissions to read Ingress resources and manage DNS records.

## Additional Resources

- [External-DNS Documentation](https://kubernetes-sigs.github.io/external-dns/)
- [Cloudflare API Token Guide](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
- [External-DNS Cloudflare Provider](https://kubernetes-sigs.github.io/external-dns/latest/configuration/providers/cloudflare/)

