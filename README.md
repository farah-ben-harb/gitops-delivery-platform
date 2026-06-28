# GitOps Delivery Platform

This repository stores the Kubernetes manifests and Argo CD application definition for the `devsecops-incident-platform` application.

## Repository Purpose

Jenkins will not deploy directly to Kubernetes.

Instead, the delivery flow will be:

1. Jenkins builds and tests the application
2. Jenkins scans the code and image
3. Jenkins pushes the image to GHCR
4. Jenkins updates the image tag in this repository
5. Argo CD detects the Git change
6. Argo CD syncs the target cluster

## Structure

```text
base/
  app-configmap.yaml
  app-deployment.yaml
  app-service.yaml
  kustomization.yaml
  namespace.yaml
  postgres-deployment.yaml
  postgres-service.yaml
  secret-example.yaml
overlays/
  local-kind/
    kustomization.yaml
argocd/
  application.yaml
```

## Current Scope

This scaffold is focused on the first local GitOps target:

- `kind` Kubernetes cluster
- FastAPI application
- PostgreSQL
- Argo CD application definition

The monitoring stack and production-hardening improvements will come in later phases.

## How Jenkins Will Update The Image

Jenkins will later update the image tag in:

- `overlays/local-kind/kustomization.yaml`

That keeps the deployment change very clear in Git history and works well with Argo CD.

## Notes About Secrets

This repository includes a committed `secret-example.yaml` with clearly fake lab values so the structure is documented.

For real environments:

- do not commit live secrets
- replace this with a safer approach such as sealed secrets or an external secret manager
- keep production credentials out of Git entirely

