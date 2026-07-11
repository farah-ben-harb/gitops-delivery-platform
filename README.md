# GitOps Delivery Platform

GitOps repository for deploying the `devsecops-incident-platform` application to Kubernetes with `Argo CD`.

This repository is the deployment and operations companion to the application repository. It stores the Kubernetes manifests, Kustomize overlays, Argo CD application definition, Prometheus scraping configuration, and Grafana dashboard provisioning.

## Repository Purpose

This repository implements a clean GitOps separation of concerns:

- the application repository builds the software artifact
- this repository defines the desired Kubernetes runtime state
- Argo CD continuously reconciles the cluster to match this Git state

Jenkins does not deploy directly to Kubernetes.

Instead, the delivery flow is:

1. Jenkins builds and tests the application
2. Jenkins runs security scans
3. Jenkins pushes the image to GHCR
4. Jenkins updates the image tag in this repository
5. Argo CD detects the Git change
6. Argo CD syncs the cluster automatically

## Target Platform

Current target:

- local `kind` Kubernetes cluster

Designed to evolve later toward:

- AKS
- EKS
- GKE

## Repository Structure

```text
argocd/
  application.yaml
base/
  app-configmap.yaml
  app-deployment.yaml
  app-service.yaml
  app-servicemonitor.yaml
  grafana-dashboard-configmap.yaml
  kustomization.yaml
  namespace.yaml
  postgres-deployment.yaml
  postgres-service.yaml
  secret-example.yaml
overlays/
  local-kind/
    kustomization.yaml
```

## What This Repository Manages

The manifests in this repository provision:

- application namespace
- FastAPI deployment
- PostgreSQL deployment
- application and database services
- non-sensitive configuration through `ConfigMap`
- example secret structure
- Prometheus `ServiceMonitor`
- Grafana dashboard ConfigMap
- Argo CD `Application`

## Deployment Architecture

High-level delivery flow:

1. Source code is updated in `devsecops-incident-platform`
2. Jenkins publishes a container image to GHCR
3. Jenkins updates `overlays/local-kind/kustomization.yaml`
4. Argo CD watches this repository
5. Argo CD syncs the new desired state to the cluster
6. Prometheus scrapes the app metrics
7. Grafana visualizes service health and incident activity

## Image Management

The overlay references the application image:

- `ghcr.io/farah-ben-harb/devsecops-incident-platform-api`

The current local-kind overlay updates the image tag here:

- `overlays/local-kind/kustomization.yaml`

Jenkins writes an immutable tag format such as:

- `sha-6526c03`

This gives:

- deterministic deployments
- traceability from running pod to source commit
- safer rollback behavior than a mutable `latest` tag

## Argo CD Application

Argo CD is configured through:

- `argocd/application.yaml`

It points to:

- repository: `https://github.com/farah-ben-harb/gitops-delivery-platform.git`
- revision: `main`
- path: `overlays/local-kind`
- destination namespace: `devsecops-platform`

Automated sync is enabled with:

- pruning
- self-healing
- namespace creation

## Kustomize Layout

The repository uses:

- `base/` for reusable manifests
- `overlays/local-kind/` for environment-specific image selection and namespace targeting

This keeps the structure ready for future overlays such as:

- `overlays/aks`
- `overlays/eks`
- `overlays/gke`

## Private GHCR Pulls

The application image is currently pulled from a private GHCR package.

The application deployment references:

- `imagePullSecrets: ghcr-pull-secret`

Create the namespace and pull secret before deployment:

```bash
kubectl create namespace devsecops-platform --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret docker-registry ghcr-pull-secret \
  --namespace devsecops-platform \
  --docker-server=ghcr.io \
  --docker-username=farah-ben-harb \
  --docker-password='<YOUR_GITHUB_PAT_WITH_READ_PACKAGES>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

The GitHub PAT for this secret should have at least:

- `read:packages`

If the package remains private and repository access is also required, include:

- `repo`

## Prometheus Scraping

The application metrics endpoint is integrated through:

- `base/app-servicemonitor.yaml`

Scrape target details:

- ServiceMonitor namespace: `devsecops-platform`
- target namespace: `devsecops-platform`
- service: `devsecops-incident-platform`
- port: `http`
- path: `/metrics`
- interval: `30s`

The application `Service` includes:

- `monitoring: enabled`

The `ServiceMonitor` selects that label so Prometheus can scrape the API metrics endpoint.

## Grafana Dashboard

The repository includes a pre-provisioned Grafana dashboard:

- `base/grafana-dashboard-configmap.yaml`

The ConfigMap carries:

- `grafana_dashboard: "1"`

This allows the `kube-prometheus-stack` Grafana sidecar to discover and load it automatically without manual dashboard import.

Dashboard name:

- `DevSecOps Incident Platform Overview`

Dashboard coverage:

- app uptime
- request throughput
- p95 latency
- API error volume
- request rate by endpoint
- latency by endpoint
- incidents by severity
- incident volume by service and severity
- request summary tables

## Bootstrap Runbook For Local kind

### 1. Create the cluster

```bash
kind create cluster --name devsecops-platform --wait 5m
kind export kubeconfig --name devsecops-platform
kubectl get nodes
```

### 2. Install Argo CD

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s
kubectl rollout status deployment/argocd-repo-server -n argocd --timeout=300s
kubectl rollout status statefulset/argocd-application-controller -n argocd --timeout=300s
```

### 3. Create the GHCR pull secret

```bash
kubectl create namespace devsecops-platform --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret docker-registry ghcr-pull-secret \
  --namespace devsecops-platform \
  --docker-server=ghcr.io \
  --docker-username=farah-ben-harb \
  --docker-password='<YOUR_GITHUB_PAT_WITH_READ_PACKAGES>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 4. Install monitoring

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
kubectl get crd servicemonitors.monitoring.coreos.com
```

### 5. Apply the Argo CD application

```bash
kubectl apply -f argocd/application.yaml
kubectl get applications -n argocd
```

## Validation Commands

Check Argo CD state:

```bash
kubectl get applications -n argocd
kubectl describe application devsecops-incident-platform -n argocd
```

Check workload state:

```bash
kubectl get all -n devsecops-platform
kubectl get pods -n monitoring
```

Open the application locally:

```bash
kubectl port-forward svc/devsecops-incident-platform -n devsecops-platform 18080:80
```

Then validate:

```bash
curl http://localhost:18080/health
curl http://localhost:18080/ready
curl http://localhost:18080/metrics
```

## Secrets Guidance

This repository includes:

- `secret-example.yaml`

It exists only to document the expected secret structure using fake placeholder values.

For real environments:

- never commit live credentials
- use a secret manager or sealed-secrets workflow
- rotate any token that was ever exposed
- keep Jenkins credentials and Kubernetes pull secrets outside Git

## Related Repository

The application source code and Jenkins pipeline live in:

- `devsecops-incident-platform`

That repository contains:

- FastAPI source code
- PostgreSQL integration
- tests
- Dockerfile
- Jenkinsfile
- metrics instrumentation
