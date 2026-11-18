# Kubernetes Two-Tier NGINX Demo

A hands-on Kubernetes project demonstrating a two-tier NGINX application deployed locally using Minikube.

The project uses Kubernetes Deployments, Services, ConfigMaps, resource management, health checks, and internal service communication.

## Architecture

```text
Browser / curl
      |
      v
Frontend Service (NodePort)
      |
      v
Frontend NGINX Deployment
      |  2 replicas
      |  Custom NGINX configuration
      |
      |  /api/ requests
      v
Backend Service (ClusterIP)
      |
      v
Backend NGINX Deployment
      |  2 replicas
      v
nginxdemos/hello
```

The frontend serves the default NGINX page at `/` and forwards requests from `/api/` to the internal backend Service.

The backend is not exposed externally.

## Technologies

- Kubernetes
- Minikube
- Docker
- NGINX
- kubectl
- YAML

## Project Structure

```text
K8s_demo/
├── README.md
├── frontend/
│   └── nginx.conf
├── k8s/
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── frontend-configmap.yaml
│   ├── frontend-deployment.yaml
│   └── frontend-service.yaml
└── evidence/
    ├── deployments.txt
    ├── pods.txt
    ├── services.txt
    ├── frontend-response.txt
    └── backend-response.txt
```

## Kubernetes Features

**Deployments and replicas**

- Two frontend replicas
- Two backend replicas
- Kubernetes manages Pod availability and replacement

**Services**

- Frontend: NodePort for external access
- Backend: ClusterIP for internal communication
- Kubernetes Service DNS enables frontend-to-backend communication

**ConfigMap**

The frontend NGINX configuration is stored in a Kubernetes ConfigMap and mounted into the frontend containers.

**Resource management**

Each container has configured CPU and memory requests and limits.

**Health checks**

Both Deployments use HTTP readiness and liveness probes.

## Prerequisites

- Docker
- Minikube
- kubectl

## Deployment

Start Minikube:

```bash
minikube start --driver=docker
```

Confirm the Kubernetes context:

```bash
kubectl config current-context
```

Deploy the manifests:

```bash
kubectl apply -f k8s/
```

Wait for the Deployments:

```bash
kubectl rollout status deployment/frontend
kubectl rollout status deployment/backend
```

## Verification

Check the Deployments:

```bash
kubectl get deployments
```

Verified deployment results:

```text
NAME       READY   UP-TO-DATE   AVAILABLE
backend    2/2     2            2
frontend   2/2     2            2
```

All four application Pods reached `Running` status with `1/1` containers ready and zero restarts during verification.

Check the Services:

```bash
kubectl get services
```

The frontend uses NodePort and the backend uses ClusterIP.

## HTTP Testing

Forward the frontend Service to localhost:

```bash
kubectl port-forward service/frontend 8080:80
```

In another terminal, test the frontend:

```bash
curl -I http://localhost:8080/
```

Test backend communication through the NGINX reverse proxy:

```bash
curl -I http://localhost:8080/api/
```

Verified results:

| Endpoint | HTTP status | Purpose |
|---|---|---|
| `/` | 200 OK | Frontend NGINX response |
| `/api/` | 200 OK | Backend response through reverse proxy |

## Deployment Evidence

Actual Minikube verification outputs are stored in the [`evidence/`](evidence/) directory.

- [Deployments](evidence/deployments.txt)
- [Pods](evidence/pods.txt)
- [Services](evidence/services.txt)
- [Frontend HTTP response](evidence/frontend-response.txt)
- [Backend HTTP response](evidence/backend-response.txt)

## Cleanup

Remove the application resources:

```bash
kubectl delete -f k8s/
```

Stop the local cluster:

```bash
minikube stop
```

## Key Learning Outcomes

- Deploying containerized applications to Kubernetes
- Configuring internal and external Kubernetes Services
- Using ConfigMaps to manage application configuration
- Implementing NGINX reverse proxy routing
- Configuring replicas, resource limits, and health probes
- Validating deployments and collecting operational evidence

