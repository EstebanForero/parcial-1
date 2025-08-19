# Application Helm Chart for advanced architectural patterns

This repository contains a Helm chart (`parcial-1`) for deploying a pedido management system with a PostgreSQL database, a Rust backend, and a React frontend. The deployment is managed using Helm for packaging and ArgoCD for continuous delivery across two environments: `dev` (`pedido-app-dev`) and `prod` (`pedido-app-prod`). This project fulfills the requirements for the "Parcial I - Patrones arquitectónicos avanzados" assignment.

## Architecture Overview

The application consists of three main components:
- **PostgreSQL**: A database for storing pedido data, deployed using the Bitnami PostgreSQL chart.
- **Backend**: A Rust Axum API, exposed via a `ClusterIP` service and accessible at `/api/*`.
- **Frontend**: A React application, exposed via a `ClusterIP` service and accessible at `/`.

These components are orchestrated using Kubernetes resources (Deployments, Services, Ingress, ConfigMap, Secret, and PersistentVolumeClaim) defined in a Helm chart. An Ingress controller (NGINX) routes traffic to the appropriate services based on the URL path. ArgoCD ensures automatic synchronization of the cluster state with the Git repository.

### Key Features Implemented
- Health Checks: Liveness and readiness probes implemented for both frontend and backend
- Horizontal Pod Autoscaler (HPA): Automatic scaling for backend based on CPU utilization
- Resource Management: CPU and memory limits/requests configured for all components
- Data Persistence: PostgreSQL with persistent volumes
- Security: Database credentials managed via Kubernetes Secrets
- GitOps: ArgoCD automated synchronization
- Multi-environment: Separate dev and prod configurations

### Architecture Diagram

<img width="786" height="642" alt="image" src="https://github.com/user-attachments/assets/6a00c574-55cf-44be-b8d8-936da18097c8" />

<img width="634" height="731" alt="image" src="https://github.com/user-attachments/assets/c41ed2e7-7a26-491f-ad21-f1b8741561a5" />

## Prerequisites

- **Kubernetes Cluster**: A running Kubernetes cluster (e.g., Minikube, Kind, or a cloud provider).
- **Helm**: Version 3.8 or later installed.
- **ArgoCD**: Installed in the cluster with access to the `argocd` namespace.
- **NGINX Ingress Controller**: Deployed in the cluster to handle Ingress resources.
- **Git**: Access to this repository (`https://github.com/EstebanForero/parcial-1`).
- **DNS Configuration** (optional for local testing): Configure `dev.parcial-1.local` (dev) for local access.

## Repository Structure

```
parcial-1/
├── Chart.yaml                      # Helm chart metadata
├── values.yaml                     # Default values
├── environments/
│   ├── dev/
│   │   ├── application-dev.yaml    # ArgoCD Application for dev
│   │   └── values-dev.yaml         # Dev-specific values
│   └── prod/
│       ├── application-prod.yaml   # ArgoCD Application for prod
│       └── values-prod.yaml        # Prod-specific values
├── templates/
│   ├── backend-deployment.yaml     # Backend Deployment with health checks
│   ├── backend-service.yaml        # Backend Service
│   ├── backend-hpa.yaml           # Backend HorizontalPodAutoscaler
│   ├── frontend-deployment.yaml   # Frontend Deployment with health checks
│   ├── frontend-service.yaml      # Frontend Service
│   ├── ingress.yaml               # Ingress for routing
│   ├── configmap.yaml             # Backend configuration (auto DB host)
│   └── secret.yaml                # Database credentials
├── .github/workflows/
│   └── deploy-helm-repo.yml       # GitHub Actions for Helm repo
└── README.md                      # This file
```

## Published Helm Chart

The Helm chart is publicly available on Artifact Hub, making it easy to search for and consume.

- **Artifact Hub Package**: [https://artifacthub.io/packages/helm/parcial-1-chart/parcial-1](https://artifacthub.io/packages/helm/parcial-1-chart/parcial-1)

## CI/CD Automation with Jenkins

This project implements a full CI/CD pipeline using Jenkins to automate the build, testing, and deployment of the backend and frontend applications. This pipeline connects the application source code repositories to the Helm deployment repository, enabling rapid and reliable releases.

### CI/CD Flow

The automation follows these key steps:

1.  **Trigger**: A developer pushes a code change to the frontend or backend application's Git repository.
2.  **CI Phase (Jenkins)**:
    *   Jenkins checks out the source code.
    *   It compiles the application and runs automated tests (for the backend).
    *   If the build and tests succeed, Jenkins builds a new Docker image, tagging it with a unique version based on the build number (e.g., `1.${BUILD_NUMBER}.0`).
    *   The newly created image is pushed to a container registry (Docker Hub).
3.  **CD Phase (Jenkins & ArgoCD)**:
    *   After a successful image push, the Jenkins pipeline automatically clones this Helm chart repository (`parcial-1`).
    *   It modifies the main `values.yaml` file, updating the `image.tag` for the corresponding component (backend or frontend) to the new version.
    *   Jenkins commits and pushes this change back to the Helm chart repository.
4.  **GitOps Synchronization (ArgoCD)**:
    *   ArgoCD, which is continuously monitoring the Helm chart repository, detects the new commit.
    *   It automatically synchronizes the cluster state, triggering a rolling update in the corresponding environment to deploy the new Docker image.

This entire process is fully automated, ensuring that only tested and verified code is deployed, and manual intervention is minimized.

### CI/CD Diagram

<img width="1170" height="417" alt="image" src="https://github.com/user-attachments/assets/5cd1fe55-26d2-49ea-9b94-32e36545f44a" />

### Working demo

<img width="804" height="453" alt="image" src="https://github.com/user-attachments/assets/38ba8e21-aa25-4e95-8646-334d8f3713ed" />

<a href="https://prod.estebanmf.space/" target="_blank" rel="noopener noreferrer">demo_page</a>

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
  -f environments/dev/values-dev.yaml \
  --namespace pedido-app-dev \
  --create-namespace
```

### Step 3: Install the Chart for Prod Environment
Deploy the application to the `pedido-app-prod` namespace:

```bash
helm install pedido-app-prod esteban-charts/parcial-1 \
  -f environments/prod/values-prod.yaml \
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

Check HPA status:
```bash
kubectl get hpa -n pedido-app-dev
kubectl get hpa -n pedido-app-prod
```

## ArgoCD Integration

ArgoCD is used for continuous deployment, automatically synchronizing the cluster with the Git repository (`https://github.com/EstebanForero/parcial-1`). It has been installed in the `argocd` namespace and is active (e.g., `argocd Active 3h36m`).

### Installation of ArgoCD (if necessary)
If ArgoCD is not already installed, deploy it using the official Helm chart:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace argocd --create-namespace
```

Access the ArgoCD UI by port-forwarding:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

Log in with the default admin password (retrieve it with `kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d`).

### Step 1: Apply ArgoCD Application Definitions
Apply the ArgoCD `Application` resources for both environments:

```bash
kubectl apply -f environments/dev/application-dev.yaml -n argocd
kubectl apply -f environments/prod/application-prod.yaml -n argocd
```

### Step 2: Verify ArgoCD Synchronization
Access the ArgoCD UI or use the CLI to check the application status:

```bash
argocd app get pedido-app-dev
argocd app get pedido-app-prod
```

The `syncPolicy.automated` settings (`prune: true`, `selfHeal: true`) ensure that any changes to the Git repository (e.g., updating `values-dev.yaml` or `values-prod.yaml`) are automatically applied to the cluster.

### Step 3: Demonstrate GitOps
To demonstrate automatic synchronization, you can manually update a value in the Git repository. For example, changing a replica count in `values-dev.yaml`.

**Note**: The process of updating the application's `image.tag` is fully automated by our Jenkins CI/CD pipeline. When a new version of the frontend or backend is built, Jenkins automatically updates the `tag` in the main `values.yaml` file and pushes the change, which ArgoCD then deploys.

To manually test the GitOps flow:
1. Update a value in the Git repository, e.g., change `frontend.replicaCount` in `environments/dev/values-dev.yaml` from `1` to `2`.
2. Commit and push the change:
   ```bash
   git commit -m "Test GitOps: Scale frontend replicas for dev"
   git push origin master
   ```
3. ArgoCD will detect the change and update the pedido-app-dev namespace automatically. Verify in the ArgoCD UI or with:
  ``` bash
  argocd app sync pedido-app-dev
  kubectl get pods -n pedido-app-dev
  ```

## DNS Configuration

### Production Domain Setup
The production environment uses an external DNS service to provide the domain `prod.estebanmf.space`. This was configured using a DNS provider (such as Cloudflare, Route53, or similar) by creating an **A record** that points to the Ingress controller's external IP address.

**Setting up your own domain:**
1. Purchase a domain from any DNS provider (Namecheap, GoDaddy, Cloudflare, etc.)
2. Get your Ingress controller's external IP: `kubectl get svc -n ingress-nginx ingress-nginx-controller`
3. Create an A record in your DNS provider:
   - **Name**: `prod` (or your preferred subdomain)
   - **Type**: `A`
   - **Value**: `<INGRESS_EXTERNAL_IP>`
   - **TTL**: `300` (or auto)

### Local Development Setup
For local testing with the dev environment, add the following to your `/etc/hosts` file:
```
<INGRESS_IP> dev.parcial-1.local
```

## Installing the Ingress Controller

An NGINX Ingress controller is required to handle Ingress resources. It is installed externally rather than as a dependency in the Helm chart, which is a better practice for the following reasons:
- **Separation of Concerns**: Keeping the Ingress controller as a separate component avoids coupling it with the application chart, making the chart more portable and reusable.
- **Cluster Management**: The Ingress controller is a cluster-wide resource, best managed independently to ensure it serves multiple applications consistently.
- **Customization**: External installation allows for tailored configuration (e.g., `LoadBalancer` service type) without altering the application chart.

### Installation Steps
Add the Ingress NGINX Helm repository and install the controller:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

Verify the installation:
```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Note the external IP of the `ingress-nginx-controller` service for DNS configuration.

## Accessing the Application

### Dev Environment
- **Frontend**: `http://dev.parcial-1.local/`
- **Backend API**: `http://dev.parcial-1.local/api`
- **Namespace**: `pedido-app-dev`

### Prod Environment  
- **Frontend**: `https://prod.estebanmf.space/`
- **Backend API**: `https://prod.estebanmf.space/api`
- **Namespace**: `pedido-app-prod`

**Note**: The production environment uses HTTPS and is accessible via the configured external domain.

## Configuration Details

### Health Checks Implementation
Both frontend and backend deployments include comprehensive health checks:

**Backend Health Checks:**
- **Liveness Probe**: Checks `/health` endpoint every 1 second after 1-second delay
- **Readiness Probe**: Checks `/health` endpoint every 5 seconds after 5-second delay
- **Port**: Uses the service port directly for reliable connectivity

**Frontend Health Checks:**
- **Liveness Probe**: Checks `/` endpoint every 3 seconds after 3-second delay  
- **Readiness Probe**: Checks `/` endpoint every 5 seconds after 3-second delay
- **Port**: Uses the service port directly for reliable connectivity

### Horizontal Pod Autoscaler (HPA)
The backend includes an HPA configuration for automatic scaling:
- **Enabled**: Controlled via `backend.autoscaling.enabled` in values files
- **Scaling Metrics**: CPU utilization percentage (configurable per environment, default is 80%)
- **Min/Max Replicas**: Configurable per environment (dev: 1-5, prod: 1-10)

### Environment-Specific Configurations

**Dev Environment (`values-dev.yaml`)**:
- Backend: 1 replica, `50m CPU`/`256Mi memory` requests, `100m CPU`/`512Mi memory` limits, image tag `latest`
- Frontend: 1 replica, image tag `1.2.0`
- PostgreSQL: 2Gi persistence
- Ingress: `dev.parcial-1.local`

**Prod Environment (`values-prod.yaml`)**:
- Backend: 3 replicas, `100m CPU`/`100Mi memory` requests, `200m CPU`/`200Mi memory` limits, image tag `latest`
- Frontend: 2 replicas, image tag `1.2.0`, `VITE_BACKEND_URL: https://prod.estebanmf.space/api`
- PostgreSQL: 10Gi persistence  
- Ingress: `prod.estebanmf.space`

### Automated Database Host Configuration
The database hostname is automatically generated using `{{ .Release.Name }}-postgresql`, eliminating the need to manually specify database hosts in environment configurations. This ensures:
- **Consistency**: Database hostnames always follow Helm conventions
- **Reduced Errors**: No risk of typos in manual configuration
- **Environment Flexibility**: Works seamlessly across dev/prod without changes

### Security Best Practices
- **Secrets Management**: Database credentials stored in Kubernetes `Secret` objects
- **Resource Limits**: All containers have defined CPU and memory limits
- **Non-root Execution**: Containers run with appropriate security contexts
- **Network Policies**: Ingress rules properly configured for service isolation

**DNS Resolution Issues:**
- Verify external IP: `kubectl get svc -n ingress-nginx`
- Check DNS propagation: `nslookup prod.estebanmf.space`
- For local testing, verify `/etc/hosts` entries

## Additional Resources and sources

- [Helm Documentation](https://helm.sh/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
