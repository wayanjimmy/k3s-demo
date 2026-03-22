# K3s Demo - GitOps with Flux CD

A demonstration repository showcasing a single-node K3s cluster managed through GitOps using Flux CD, complete with SOPS-based secret encryption.

## About

This repository is the practical artifact of the tutorial:

**[Setup K3s Single Node Cluster, GitOps dengan Flux](https://blog.wayanjim.my.id/en/posts/setup-k3s-single-node-gitops-dengan-flux)**

The blog post provides a comprehensive walkthrough of setting up a homelab-ready Kubernetes cluster using K3s + Lima-VM, managed entirely through GitOps with Flux CD.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Git Repository                       │
│                      (Single Source of Truth)                │
└────────────────────────────┬────────────────────────────────┘
                             │ Flux (periodic sync)
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    K3s Single Node Cluster                  │
│                    (via Lima-VM)                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           flux-system namespace                       │  │
│  │  • source-controller                                 │  │
│  │  • kustomize-controller                              │  │
│  │  • helm-controller                                   │  │
│  │  • notification-controller                           │  │
│  │  • image-reflector-controller                        │  │
│  │  • image-automation-controller                       │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           demo-apps namespace                        │  │
│  │  • nginx-demo (Deployment + Service)                │  │
│  │  • db-credentials (SOPS encrypted secret)           │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

- [Lima-VM](https://lima-vm.io/) - Linux virtual machines on macOS
- [kubectl](https://kubernetes.io/docs/tasks/tools/) - Kubernetes command-line tool
- [flux](https://fluxcd.io/docs/installation/) - Flux CLI
- [sops](https://github.com/getsops/sops) - Secrets encryption
- [age](https://github.com/FiloSottile/age) - Modern encryption tool
- [direnv](https://direnv.net/) - Environment variable management (optional but recommended)

> **Tip:** You can install most tools using [arkade](https://github.com/alexellis/arkade):
> ```bash
> arkade get kubectl flux sops
> ```

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/wayanjimmy/k3s-demo.git
cd k3s-demo
```

### 2. Set Up Lima-VM with K3s

```bash
limactl create template:k3s \
    --name=k3s-demo \
    --cpus=2 \
    --memory=2 \
    --disk=20 \
    --yes
```

### 3. Configure kubectl

Set up your environment using direnv:

```bash
cp .envrc.example .envrc
# Edit .envrc with your KUBECONFIG path if different
direnv allow
```

Or manually export:

```bash
export KUBECONFIG="$HOME/.lima/k3s-demo/copied-from-guest/kubeconfig.yaml"
```

Verify cluster connection:

```bash
kubectl cluster-info
```

### 4. Bootstrap Flux

Make sure your SSH key is registered with GitHub, then bootstrap Flux:

```bash
flux bootstrap git \
    --url=ssh://git@github.com/wayanjimmy/k3s-demo.git \
    --private-key-file=$HOME/.ssh/id_ed25519 \
    --branch=main \
    --path=clusters/k3s-demo \
    --components-extra image-reflector-controller,image-automation-controller
```

### 5. Verify Installation

Check that Flux is running correctly:

```bash
flux check
kubectl get pods -n flux-system
```

## Repository Structure

```
.
├── clusters/
│   └── k3s-demo/
│       ├── demo-apps-kustomization.yaml    # Flux Kustomization definition
│       ├── flux-system/                    # Flux system manifests
│       │   ├── kustomization.yaml
│       │   ├── gotk-components.yaml
│       │   └── gotk-sync.yaml
│       └── demo-apps/                      # Application manifests
│           ├── kustomization.yaml          # Kustomize resources
│           ├── namespace.yaml
│           ├── nginx-demo.yaml             # Nginx deployment + service
│           └── secret.yaml                 # SOPS encrypted secret
├── secrets/
│   └── .gitignore                          # Prevents committing private keys
├── .envrc.example                          # Environment variable template
└── README.md
```

## Secret Management with SOPS

This repository demonstrates how to manage secrets securely using SOPS + Age encryption.

### Generate Age Keys

```bash
age-keygen -o secrets/flux-age-key.txt
```

This generates a key pair with:
- **Public key** (can be shared in the repo)
- **Private key** (stored as Kubernetes secret)

### Create Kubernetes Secret for Decryption

```bash
kubectl create secret generic sops-age \
    --namespace=flux-system \
    --from-file=age.agekey=$(pwd)/secrets/flux-age-key.txt
```

### Encrypt Secrets

First, create a plain secret file:

```yaml
# clusters/k3s-demo/demo-apps/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: demo-apps
type: Opaque
stringData:
  DB_HOST: postgres.demo-apps.svc.cluster.local
  DB_PORT: "5432"
  DB_NAME: mydb
  DB_USER: admin
  DB_PASSWORD: secret
```

Then encrypt it with SOPS:

```bash
export SOPS_PUBLIC_KEY="<your-age-public-key>"

sops --encrypt \
    --age "$SOPS_PUBLIC_KEY" \
    --encrypted-regex '^(data|stringData)$' \
    --in-place clusters/k3s-demo/demo-apps/secret.yaml
```

### Decryption in Cluster

The `demo-apps-kustomization.yaml` is configured to use the `sops-age` secret for decryption:

```yaml
decryption:
  provider: sops
  secretRef:
    name: sops-age
```

Flux automatically decrypts secrets when reconciling manifests.

## GitOps Workflow

With Flux installed, deploying changes is as simple as:

1. **Edit** manifests in the `clusters/k3s-demo/` directory
2. **Commit** your changes
3. **Push** to the repository

Flux will automatically reconcile the cluster state within the defined interval (1 minute).

```bash
git add .
git commit -m "feat: add new application"
git push origin main
```

Monitor reconciliation:

```bash
# Watch kustomizations
flux get kustomizations -n flux-system --watch

# Check specific kustomization
flux get kustomization demo-apps -n flux-system
```

## Testing

### Access the Demo Application

The nginx-demo service is exposed via NodePort on port 30081:

```bash
# From Lima VM host
curl http://localhost:30081

# Or port-forward
kubectl port-forward -n demo-apps svc/nginx-demo 8080:80
curl http://localhost:8080
```

### Verify Secret Decryption

Check the initContainer logs to confirm secrets are properly decrypted:

```bash
kubectl logs -f -l app=nginx-demo -c secret-test -n demo-apps
```

Expected output:

```
=== SOPS Secret Decryption Test ===
DB_HOST: postgres.demo-apps.svc.cluster.local
```

## Troubleshooting

### Flux Issues

```bash
# Check Flux status
flux check

# View reconciliation logs
flux logs -n flux-system --all

# Get kustomization status
flux get kustomizations -n flux-system
```

### Secret Decryption Issues

```bash
# Verify sops-age secret exists
kubectl get secret sops-age -n flux-system

# Check public key in encrypted secret
sops --decrypt clusters/k3s-demo/demo-apps/secret.yaml
```

### Pod Issues

```bash
# Describe pod
kubectl describe pod -n demo-apps -l app=nginx-demo

# View logs
kubectl logs -n demo-apps -l app=nginx-demo --all-containers
```

## Why Flux?

This project uses Flux over alternatives like ArgoCD because of:

- **Native SOPS integration** - Seamless secret encryption/decryption
- **Image automation** - Automatic deployment when new images are pushed
- **Git-centric workflow** - Simple, declarative configuration management
- **Toolchain flexibility** - Works well with Kustomize, Helm, and plain manifests

## Links

- [Blog Post - Setup K3s Single Node GitOps dengan Flux](https://blog.wayanjim.my.id/en/posts/setup-k3s-single-node-gitops-dengan-flux)
- [Flux Documentation](https://fluxcd.io/docs/)
- [SOPS Documentation](https://github.com/getsops/sops)
- [K3s Documentation](https://docs.k3s.io/)
- [Lima-VM Documentation](https://lima-vm.io/)
