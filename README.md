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

Jenkins will set the deployment to an immutable tag format like:

- `sha-6526c03`

That means the cluster always pulls a specific build artifact instead of a mutable `latest` tag.

## Private GHCR Pulls

The application image is pulled from a private GHCR package:

- `ghcr.io/farah-ben-harb/devsecops-incident-platform-api`

Because of that, the application deployment references:

- `imagePullSecrets: ghcr-pull-secret`

Before deploying to the cluster, create that secret in the target namespace:

```bash
kubectl create namespace devsecops-platform --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret docker-registry ghcr-pull-secret \
  --namespace devsecops-platform \
  --docker-server=ghcr.io \
  --docker-username=farah-ben-harb \
  --docker-password='<YOUR_GITHUB_PAT_WITH_READ_PACKAGES>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

The PAT used for this secret should have at least:

- `read:packages`

## Prometheus Scraping

The repository includes a `ServiceMonitor` so the Prometheus Operator from `kube-prometheus-stack` can scrape the application metrics endpoint.

The target configuration is:

- ServiceMonitor namespace: `devsecops-platform`
- Target namespace: `devsecops-platform`
- Service: `devsecops-incident-platform`
- Port: `http`
- Path: `/metrics`
- Interval: `30s`

The `ServiceMonitor` lives in:

- `base/app-servicemonitor.yaml`

The app `Service` carries:

- `monitoring: enabled`

and the `ServiceMonitor` selects that label.

## Notes About Secrets

This repository includes a committed `secret-example.yaml` with clearly fake lab values so the structure is documented.

For real environments:

- do not commit live secrets
- replace this with a safer approach such as sealed secrets or an external secret manager
- keep production credentials out of Git entirely
