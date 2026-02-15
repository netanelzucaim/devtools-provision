# Gitea Helm Chart

This directory contains a Helm chart for deploying Gitea, a painless self-hosted Git service.

## Overview

Gitea is a lightweight Git service that provides:
- Repository hosting
- Issue tracking
- Pull requests
- Wiki
- Webhooks
- User management
- Organizations and teams
- And much more!

## Quick Start

See [INSTALL.md](./INSTALL.md) for complete installation instructions.

### Prerequisites

- Kubernetes cluster (tested with Kind)
- Helm 3.x
- kubectl

### Install

```bash
cd devtools/gitea
helm install gitea . --namespace gitea --create-namespace
```

### Access

```bash
kubectl port-forward svc/gitea-http -n gitea 3000:3000
```

Then open http://localhost:3000 in your browser.

## Configuration

The chart is configured through `values.yaml`. Key configurations:

- **Admin User**: Default username is `gitea_admin` with password `changeme` (change this!)
- **Database**: SQLite3 for simplicity
- **Storage**: 10Gi persistent volume (configurable)
- **Resources**: 100m CPU / 128Mi RAM requests, 500m CPU / 512Mi RAM limits

## Architecture

This is a self-contained Helm chart with the following components:

- **StatefulSet**: Runs the Gitea application
- **Services**: HTTP (port 3000) and SSH (port 22)
- **PersistentVolume**: Stores Git repositories and database
- **ConfigMap**: Application configuration

## Security Notes

⚠️ **Important**: This configuration is optimized for local development. For production:

1. Change the default admin password
2. Set explicit SECRET_KEY and INTERNAL_TOKEN
3. Enable TLS/HTTPS
4. Configure proper authentication
5. Review security settings in values.yaml

## Customization

Edit `values.yaml` to customize:

- Image version
- Resource limits
- Persistence settings
- Gitea configuration
- Admin credentials

## Documentation

- [Installation Guide](./INSTALL.md) - Complete setup instructions
- [Gitea Documentation](https://docs.gitea.io/) - Official Gitea docs
- [values.yaml](./values.yaml) - Configuration reference

## Version

- **Chart Version**: 1.0.0
- **Gitea Version**: 1.22.0

## License

This Helm chart follows the same license as Gitea (MIT).
