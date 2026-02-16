# DevTools Provision

A comprehensive repository for provisioning and configuring development tools and infrastructure using GitOps principles.

## Overview

This repository contains configuration files, Helm charts, and setup guides for various DevOps and development tools. The goal is to provide a streamlined, reproducible way to set up development environments and infrastructure components using declarative configurations.

## Tools Included

### Argo CD

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes. This repository includes a complete Helm-based setup for running Argo CD on a Kind (Kubernetes in Docker) cluster.

**Version Information:**
- **Argo CD Application:** v2.13.0
- **Helm Chart:** 7.7.5

**Features:**
- Optimized for local Kind cluster development
- Accessible via port-forward on localhost:8080
- Pre-configured with resource limits suitable for development
- GitHub repository integration support
- No TLS configuration required (development mode)
- Works on any network without IP address concerns

📚 **[Complete Argo CD Installation Guide →](devtools/argocd/INSTALL.md)**

### Woodpecker CI

Woodpecker CI is a simple, lightweight CI engine with great extensibility. This repository includes a Helm-based setup for running Woodpecker CI on a Kind cluster.

**Version Information:**
- **Woodpecker CI Application:** v2.7.0
- **Helm Chart:** 1.6.0

**Features:**
- Lightweight and easy to set up
- Accessible via port-forward on localhost:9000
- GitHub OAuth integration support
- Built-in agent support with 2 replicas
- SQLite database for local development
- Persistent storage for CI data
- Works on any network using port-forward

📚 **[Complete Woodpecker CI Installation Guide →](devtools/woodpecker/INSTALL.md)**

## Quick Start

### Prerequisites

Ensure you have the following installed:
- Docker (20.10+)
- Kind (0.20.0+)
- kubectl (1.27.0+)
- Helm 3.x (3.12.0+)

### Install Argo CD on Kind

```bash
# 1. Create a Kind cluster
kind create cluster --name argocd-cluster

# 2. Navigate to the Argo CD charts directory
cd devtools/argocd/charts

# 3. Add Argo CD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 4. Update Helm dependencies
helm dependency update

# 5. Install Argo CD
kubectl create namespace argocd
helm install argocd . --namespace argocd --wait

# 6. Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# 7. Set up port-forward to access the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 8. Access the UI at http://localhost:8080
# Login with username: admin and the password from step 6
```

For detailed instructions, troubleshooting, and advanced configuration, see the [Argo CD Installation Guide](devtools/argocd/INSTALL.md).

### Install Woodpecker CI on Kind

```bash
# 1. Create a Kind cluster (or reuse existing)
kind create cluster --name woodpecker-cluster

# 2. Navigate to the Woodpecker CI charts directory
cd devtools/woodpecker/charts

# 3. Install Woodpecker CI
kubectl create namespace woodpecker
helm install woodpecker . --namespace woodpecker --wait

# 4. Set up port-forward to access the UI
kubectl port-forward svc/woodpecker-server -n woodpecker 9000:8000

# 5. Access the UI at http://localhost:9000
# Register as a new user or configure GitHub OAuth
```

For detailed instructions, troubleshooting, and GitHub OAuth setup, see the [Woodpecker CI Installation Guide](devtools/woodpecker/INSTALL.md).

## Repository Structure

```
.
├── application.yaml         # Argo CD Application to deploy the ApplicationSet
├── applicationset.yaml      # ApplicationSet for GitOps auto-deployment
└── devtools/
    ├── argocd/
    │   ├── charts/
    │   │   ├── Chart.yaml       # Helm chart definition with dependencies
    │   │   └── values.yaml      # Custom configuration values for Argo CD
    │   └── INSTALL.md           # Comprehensive installation guide
    └── woodpecker/
        ├── charts/
        │   ├── Chart.yaml       # Helm chart definition (standalone)
        │   ├── values.yaml      # Custom configuration values for Woodpecker CI
        │   └── templates/       # Kubernetes manifests templates
        │       ├── agent-deployment.yaml
        │       ├── server-deployment.yaml
        │       ├── server-pvc.yaml
        │       └── server-service.yaml
        └── INSTALL.md           # Comprehensive installation guide
```

## GitOps Automation with ApplicationSet

This repository includes an ApplicationSet that automatically discovers and deploys all tools in the `devtools/` directory using GitOps principles.

### How It Works

1. **Auto-Discovery:** The ApplicationSet scans the `devtools/` directory for subdirectories containing a `charts/` folder
2. **Automatic Creation:** For each discovered tool, it automatically creates an Argo CD Application
3. **Self-Healing:** Applications are configured with automated sync, prune, and self-heal policies
4. **Namespace Management:** Each tool is deployed to its own namespace (automatically created)

### Using the ApplicationSet

Once Argo CD is installed, deploy the ApplicationSet to enable automatic management of all devtools:

```bash
# Option 1: Bootstrap by applying the Application that deploys the ApplicationSet
kubectl apply -f application.yaml

# Option 2: Apply the ApplicationSet directly
kubectl apply -f applicationset.yaml

# Verify ApplicationSet is created
kubectl get applicationset -n argocd

# View auto-generated applications
kubectl get applications -n argocd
```

The ApplicationSet will automatically:
- Deploy Argo CD to the `argocd` namespace
- Deploy Woodpecker CI to the `woodpecker` namespace
- Deploy any future tools added to `devtools/` with a `charts/` folder

**Recommended Approach:** Use `application.yaml` to bootstrap the ApplicationSet. This creates an Argo CD Application that manages the ApplicationSet itself, providing full GitOps workflow for the entire devtools infrastructure.

### Benefits

- **Zero Manual Configuration:** New tools are automatically discovered and deployed
- **Consistent Deployment:** All tools follow the same deployment pattern
- **GitOps Native:** Changes to tool configurations in Git are automatically synced
- **Easy Maintenance:** Update tool versions by modifying `Chart.yaml` or `values.yaml`

### Adding New Tools

To add a new tool to the automated deployment:

1. Create a new directory in `devtools/` (e.g., `devtools/newtool/`)
2. Create a `charts/` subdirectory with `Chart.yaml` and `values.yaml`
3. Commit and push to the repository
4. The ApplicationSet will automatically detect and deploy the new tool

## GitOps Workflow

Once Argo CD is installed, you can use it to manage applications declaratively:

1. **Define applications** in Git repositories using Kubernetes manifests, Helm charts, or Kustomize
2. **Configure Argo CD applications** to point to your Git repositories (or use the ApplicationSet for automation)
3. **Let Argo CD sync** your cluster state with the desired state in Git
4. **Monitor and manage** applications through the Argo CD UI or CLI

## Contributing

Contributions are welcome! If you have improvements, additional tools, or better configurations, please feel free to open an issue or submit a pull request.

## License

This repository is open source and available for use in your projects.

## Additional Resources

- [Argo CD Official Documentation](https://argo-cd.readthedocs.io/)
- [Woodpecker CI Official Documentation](https://woodpecker-ci.org/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kind Documentation](https://kind.sigs.k8s.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)