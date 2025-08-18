# Pedido Application Helm Chart

This repository contains a Helm chart (`parcial-1`) for deploying a pedido management system with a PostgreSQL database, a Java Spring Boot backend, and a React frontend. The deployment is managed using Helm for packaging and ArgoCD for continuous delivery across two environments: `dev` (`pedido-app-dev`) and `prod` (`pedido-app-prod`). This project fulfills the requirements for the "Parcial I - Patrones arquitectónicos avanzados" assignment.

## Architecture Overview

The application consists of three main components:
- **PostgreSQL**: A database for storing pedido data, deployed using the Bitnami PostgreSQL chart.
- **Backend**: A Java Spring Boot API, exposed via a `ClusterIP` service and accessible at `/api/*`.
- **Frontend**: A React application, exposed via a `ClusterIP` service and accessible at `/`.

These components are orchestrated using Kubernetes resources (Deployments, Services, Ingress, ConfigMap, Secret, and PersistentVolumeClaim) defined in a Helm chart. An Ingress controller (NGINX) routes traffic to the appropriate services based on the URL path. ArgoCD ensures automatic synchronization of the cluster state with the Git repository.

### Architecture Diagram

<img width="807" height="666" alt="image" src="https://github.com/user-attachments/assets/5aebf619-8454-4d75-9649-a9af98a03635" />

```
+-------------------+       +-------------------+       +-------------------+
| Ingress (NGINX)   |       |                   |       |                   |
| - /api/* -> Backend |---->| Backend (Spring)  |<----->| PostgreSQL (Bitnami) |
| - / -> Frontend   |       |                   |       |                   |
+-------------------+       +-------------------+       +-------------------+
          |                        |
          |                        |
          v                        v
+-------------------+       +-------------------+
| Frontend (React)  |       |                   |
|                   |       |                   |
+-------------------+       +-------------------+
```

## Prerequisites

- **Kubernetes Cluster**: A running Kubernetes cluster (e.g., Minikube, Kind, or a cloud provider).
- **Helm**: Version 3.8 or later installed.
- **ArgoCD**: Installed in the cluster with access to the `argocd` namespace.
- **NGINX Ingress Controller**: Deployed in the cluster to handle Ingress resources.
- **Git**: Access to this repository (`https://github.com/EstebanForero/parcial-1`).
- **DNS Configuration** (optional for local testing): Configure `dev.parcial-1.local` (dev) and `prod.estebanmf.space` (prod) to resolve to the Ingress controller’s IP.

## Repository Structure

```
parcial-1/
├── charts/
│   └── pedido-app/
│       ├── Chart.yaml              # Helm chart metadata
│       ├── values.yaml             # Default values
│       ├── environments/
│       │   ├── dev/
│       │   │   ├── application-dev.yaml  # ArgoCD Application for dev
│       │   │   ├── values-dev.yaml      # Dev-specific values
│       │   ├── prod/
│       │   │   ├── application-prod.yaml # ArgoCD Application for prod
│       │   │   ├── values-prod.yaml     # Prod-specific values
│       ├── templates/
│       │   ├── backend-deployment.yaml  # Backend Deployment
│       │   ├── backend-service.yaml     # Backend Service
│       │   ├── backend-hpa.yaml         # Backend HorizontalPodAutoscaler
│       │   ├── frontend-deployment.yaml # Frontend Deployment
│       │   ├── frontend-service.yaml    # Frontend Service
│       │   ├── ingress.yaml             # Ingress for routing
│       │   ├── configmap.yaml          # Backend configuration
│       │   ├── secret.yaml             # Database credentials
├── README.md                           # This file
```

## Installation with Helm (Manual)

### Step 1: Add the Helm Repository
The Helm chart is hosted at `https://EstebanForero.github.io/parcial-1`.

```bash
helm repo add esteban-charts https://EstebanForero.github.io/parcial-1
helm repo update
```

Verify the chart is available:
```bash
helm search repo esteban-charts
# Output:
# NAME                     CHART VERSION APP VERSION DESCRIPTION
# esteban-charts/parcial-1 0.1.0         1.0.0       A Helm chart for deploying the frontend and backend...
```

### Step 2: Install the Chart for Dev Environment
Deploy the application to the `pedido-app-dev` namespace:

```bash
helm install pedido-app-dev esteban-charts/parcial-1 \
  -f charts/pedido-app/environments/dev/values-dev.yaml \
  --namespace pedido-app-dev \
  --create-namespace
```

### Step 3: Install the Chart for Prod Environment
Deploy the application to the `pedido-app-prod` namespace:

```bash
helm install pedido-app-prod esteban-charts/parcial-1 \
  -f charts/pedido-app/environments/prod/values-prod.yaml \
  --namespace pedido-app-prod \
  --create-namespace
```

### Step 4: Verify Deployment
Check the status of the pods in each namespace:

```bash
kubectl get pods -n pedido-app-dev
kubectl get pods -n pedido-app-prod
```

Verify the Ingress resources:
```bash
kubectl get ingress -n pedido-app-dev
kubectl get ingress -n pedido-app-prod
```

## ArgoCD Configuration

ArgoCD is used for continuous deployment, automatically synchronizing the cluster with the Git repository (`https://github.com/EstebanForero/parcial-1`).

### Step 1: Apply ArgoCD Application Definitions
Apply the ArgoCD `Application` resources for both environments:

```bash
kubectl apply -f charts/pedido-app/environments/dev/application-dev.yaml -n argocd
kubectl apply -f charts/pedido-app/environments/prod/application-prod.yaml -n argocd
```

### Step 2: Verify ArgoCD Synchronization
Access the ArgoCD UI or use the CLI to check the application status:

```bash
argocd app get pedido-app-dev
argocd app get pedido-app-prod
```

The `syncPolicy.automated` settings (`prune: true`, `selfHeal: true`) ensure that any changes to the Git repository (e.g., updating `values-dev.yaml` or `values-prod.yaml`) are automatically applied to the cluster.

### Step 3: Demonstrate GitOps
To demonstrate automatic synchronization:
1. Update a value in the Git repository, e.g., change `backend.image.tag` in `charts/pedido-app/environments/dev/values-dev.yaml` from `"1.13.0"` to `"1.14.0"`.
2. Commit and push the change:
   ```bash
   git commit -m "Update backend image tag for dev"
   git push origin master
   ```
3. ArgoCD will detect the change and update the `pedido-app-dev` namespace automatically. Verify in the ArgoCD UI or with:
   ```bash
   argocd app sync pedido-app-dev
   kubectl get pods -n pedido-app-dev
   ```

## Accessing the Application

### Dev Environment
- **Frontend**: `http://dev.parcial-1.local/`
- **Backend API**: `http://dev.parcial-1.local/api`
- **Namespace**: `pedido-app-dev`

### Prod Environment
- **Frontend**: `https://prod.estebanmf.space/`
- **Backend API**: `https://prod.estebanmf.space/api`
- **Namespace**: `pedido-app-prod`

**Note**: Ensure the Ingress controller is properly configured and the domain names resolve to the controller’s IP. For local testing, update your `/etc/hosts` file or equivalent:
```
<INGRESS_IP> dev.parcial-1.local
<INGRESS_IP> prod.estebanmf.space
```

## Configuration Details

### Helm Chart
- **Chart Name**: `parcial-1`
- **Version**: `0.1.0`
- **App Version**: `1.0.0`
- **Dependencies**: Bitnami PostgreSQL (`16.7.26`)
- **Templates**:
  - `backend-deployment.yaml`: Deploys the Spring Boot backend.
  - `backend-service.yaml`: Exposes the backend via `ClusterIP`.
  - `backend-hpa.yaml`: Autoscales the backend based on CPU usage (min: 1, max: 5, target: 80%).
  - `frontend-deployment.yaml`: Deploys the React frontend.
  - `frontend-service.yaml`: Exposes the frontend via `ClusterIP`.
  - `ingress.yaml`: Routes traffic to backend (`/api/*`) and frontend (`/`).
  - `configmap.yaml`: Stores non-sensitive backend configurations (e.g., DB host, port).
  - `secret.yaml`: Stores the PostgreSQL password securely.

### Environment-Specific Configurations
- **Dev (`values-dev.yaml`)**:
  - Backend: 1 replica, `100m CPU`/`256Mi memory` requests, `500m CPU`/`512Mi memory` limits, image tag `1.13.0`.
  - Frontend: 1 replica, image tag `1.2.0`.
  - PostgreSQL: 2Gi persistence.
  - Ingress: `dev.parcial-1.local`.
- **Prod (`values-prod.yaml`)**:
  - Backend: 3 replicas, `100m CPU`/`100Mi memory` requests, `200m CPU`/`200Mi memory` limits, image tag `1.13.0`.
  - Frontend: 2 replicas, image tag `1.2.0`, `VITE_BACKEND_URL: https://prod.estebanmf.space/api`.
  - PostgreSQL: 10Gi persistence.
  - Ingress: `prod.estebanmf.space`.

### Good Practices
- **Resource Management**: CPU and memory `requests` and `limits` are defined for both backend and frontend, tailored per environment.
- **Security**: Database credentials are stored in a Kubernetes `Secret`.
- **Persistence**: PostgreSQL uses a `PersistentVolumeClaim` to ensure data persistence.
- **Modularity**: The Helm chart is parameterized, with separate `values.yaml` files for dev and prod.
- **Autoscaling**: The backend includes a HorizontalPodAutoscaler (`backend-hpa.yaml`).
- **GitOps**: ArgoCD automates deployment and synchronization, ensuring changes in Git are reflected in the cluster.

## Troubleshooting

- **Pods not starting**: Check pod logs with `kubectl logs <pod-name> -n <namespace>`.
- **Ingress not working**: Ensure the NGINX Ingress controller is running and the domain resolves correctly (`kubectl get ingress -n <namespace>`).
- **ArgoCD sync issues**: Verify the repository URL and credentials in `application-dev.yaml`/`application-prod.yaml`. Check ArgoCD logs: `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server`.
- **Helm installation errors**: Ensure the Bitnami repository is added (`helm repo add bitnami https://charts.bitnami.com/bitnami`) and the chart version matches (`16.7.26`).

## License
This project is for academic purposes as part of the "Patrones arquitectónicos avanzados" course. No specific license is applied.
