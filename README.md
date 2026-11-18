# K8s Demo — Two-Tier NGINX Deployment

A Kubernetes deployment demo running two NGINX-based services as separate frontend and bac>

## What This Does

Deploys a two-tier architecture using public Docker images and standard Kubernetes primiti>

- **Frontend** — `nginx:alpine`, exposed externally via NodePort
- **Backend** — `nginxdemos/hello`, accessible internally via ClusterIP

## Repo Structure

```
k8s/
├── frontend-deployment.yaml   # NGINX alpine, 1 replica
├── frontend-service.yaml      # NodePort — external access
├── backend-deployment.yaml    # nginxdemos/hello, 1 replica
└── backend-service.yaml       # ClusterIP — internal only
```

## Prerequisites

- A running Kubernetes cluster (Minikube, K3s, or cloud-managed)
- `kubectl` configured and pointing at your cluster

## Deploy

```bash
kubectl apply -f k8s/
```

## Verify

```bash
kubectl get deployments
kubectl get services
kubectl get pods
```

## Access the Frontend

```bash
# Get the NodePort


