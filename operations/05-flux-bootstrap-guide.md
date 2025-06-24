# Flux CD Bootstrap Guide

|             |                           |
| ----------- | ------------------------- |
| **Version** | 1.0                       |
| **Date**    | 2025-06-17                |
| **Owner**   | Cloud Infrastructure Team |

This guide provides step-by-step instructions for bootstrapping Flux CD on Kubernetes clusters for the Piksel Digital Earth project.

## Prerequisites

### 1. Required Tools

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Verify installation
flux --version
```

### 2. Kubernetes Cluster

- Kubernetes cluster running
- `kubectl` configured and connected to the cluster
- Cluster admin permissions

How to configure kubectl and connect to piksel's AWS Clusters: [Connect-to-EKS-Guide](./04-Connect-to-EKS-cluster-config-guide.md)

### 3. GitHub Repository Access

- GitHub repository: `piksel-ina/piksel-gitops`
- Write permissions to the repository

## SSH Key Setup

### Generate Dedicated SSH Key

Each environment requires a dedicated SSH key for security isolation:

```bash
# For development environment
ssh-keygen -t ed25519 -f ~/.ssh/flux-development-key -C "flux-development-deploy"

# For production environment
ssh-keygen -t ed25519 -f ~/.ssh/flux-production-key -C "flux-production-deploy"

# For staging environment (if needed)
ssh-keygen -t ed25519 -f ~/.ssh/flux-staging-key -C "flux-staging-deploy"
```

### Add Deploy Key to GitHub

1. Copy the public key content:

```bash
cat ~/.ssh/flux-development-key.pub
```

2. Navigate to GitHub repository: `piksel-ina/piksel-gitops`
3. Go to **Settings** → **Deploy keys**
4. Click **Add deploy key**
5. Paste the public key content
6. **Enable "Allow write access"** checkbox
7. Set title: `flux-development-deploy`

Repeat this process for each environment with the respective public key.

## Bootstrap Flux

### Step 1: Verify Prerequisites

```bash
# Check cluster connectivity
kubectl cluster-info

# Check Flux prerequisites
flux check --pre

# Test SSH key access
ssh -T git@github.com -i ~/.ssh/flux-development-key

# Check if the key is loaded
ssh-add -l

# Add key if needed
ssh-add ~/.ssh/flux-development-key
```

### Step 2: Run Bootstrap Command

Execute the bootstrap command for the target environment.

e.g, development environment:

```bash
flux bootstrap github \
  --owner=piksel-ina \
  --repository=piksel-gitops \
  --branch=main \
  --path=clusters/development \
  --read-write-key \
  --private-key-file=~/.ssh/flux-development-key
```

or,

```bash
flux bootstrap git \
  --url=ssh://git@github.com/piksel-ina/piksel-gitops \
  --branch=main \
  --path=clusters/development
```

The `--read-write-key` flag ensures Flux has full GitOps capabilities including:

- Image automation and updates
- Automated manifest commits
- Release management
- Full GitOps workflow support

### Step 3: Verify Bootstrap Success

```bash
# Check Flux system components
kubectl get pods -n flux-system

# Verify GitRepository sync
flux get sources git

# Check Kustomizations status
flux get kustomizations

# Verify detailed status
kubectl get gitrepositories -n flux-system
kubectl get kustomizations -n flux-system
```

## Expected Output

After successful bootstrap, the following output should be visible:

```bash
$ flux get sources git
NAME            REVISION                SUSPENDED       READY   MESSAGE
flux-system     main@sha1:xxxxxxx       False           True    stored artifact for revision 'main@sha1:xxxxxxx'

$ flux get kustomizations
NAME            REVISION                SUSPENDED       READY   MESSAGE
flux-system     main@sha1:xxxxxxx       False           True    Applied revision: main@sha1:xxxxxxx
```

## Repository Structure

After bootstrap, the repository structure will include:

```
piksel-gitops/
├── clusters/
│   └── {environment}/
│       └── flux-system/
│           ├── gotk-components.yaml     # Flux components
│           ├── gotk-sync.yaml           # Sync configuration
│           └── kustomization.yaml       # Kustomization manifest
```

## Security Best Practices

1. **Dedicated Keys**: Use separate SSH keys for each environment
2. **Key Rotation**: Regularly rotate SSH keys (quarterly recommended)
3. **Access Monitoring**: Monitor repository access logs
4. **Principle of Least Privilege**: Each key should only access necessary repositories
5. **Secure Storage**: Store private keys securely and never commit them to version control

## Support Resources

- **Flux CD Documentation**: https://fluxcd.io/docs/
- **Repository Issues**: Create issues in the `piksel-ina/piksel-gitops` repository
- **Troubleshooting Guide**: Refer to `docs/troubleshooting.md`
