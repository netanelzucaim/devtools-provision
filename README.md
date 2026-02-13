# DevTools Provision

A comprehensive repository for provisioning and configuring development tools and infrastructure using GitOps principles.

## Overview

This repository contains configuration files, Helm charts, and setup guides for various DevOps and development tools. The goal is to provide a streamlined, reproducible way to set up development environments and infrastructure components using declarative configurations.

## Tools Included

### Argo CD

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes. This repository includes a complete Helm-based setup for running Argo CD on a Kind (Kubernetes in Docker) cluster.

**Features:**
- Optimized for local Kind cluster development
- Accessible via port-forward on localhost:8080
- Pre-configured with resource limits suitable for development
- GitHub repository integration support
- No TLS configuration required (development mode)
- Works on any network without IP address concerns

📚 **[Complete Argo CD Installation Guide →](devtools/argocd/INSTALL.md)**

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

## Repository Structure

```
devtools/
└── argocd/
    ├── charts/
    │   ├── Chart.yaml       # Helm chart definition with dependencies
    │   └── values.yaml      # Custom configuration values for Argo CD
    └── INSTALL.md           # Comprehensive installation guide
```

## GitOps Workflow

Once Argo CD is installed, you can use it to manage applications declaratively:

1. **Define applications** in Git repositories using Kubernetes manifests, Helm charts, or Kustomize
2. **Configure Argo CD applications** to point to your Git repositories
3. **Let Argo CD sync** your cluster state with the desired state in Git
4. **Monitor and manage** applications through the Argo CD UI or CLI

## Contributing

Contributions are welcome! If you have improvements, additional tools, or better configurations, please feel free to open an issue or submit a pull request.

## License

This repository is open source and available for use in your projects.

## Additional Resources

- [Argo CD Official Documentation](https://argo-cd.readthedocs.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kind Documentation](https://kind.sigs.k8s.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)