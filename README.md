# GitOps Delivery Platform

This repository will store the Kubernetes manifests and Argo CD application definitions for the `devsecops-incident-platform` project.

## Planned Contents

- `namespace.yaml`
- `deployment.yaml`
- `service.yaml`
- `configmap.yaml`
- `secret-example.yaml`
- Argo CD application manifest
- local `kind` overlays

The repository is intentionally minimal in Phase 1 because the current focus is getting the FastAPI application working locally before we move into Kubernetes and GitOps delivery.
