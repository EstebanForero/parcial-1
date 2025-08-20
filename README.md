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

## Security and Secrets Management

This Helm chart implements a robust security model for managing database credentials, supporting two distinct modes to accommodate different environments.

### Mode 1: Auto-Generated Secret (Default for Dev/Testing)

By default, if no existing secret is specified, the chart will automatically create a Kubernetes Secret for the backend. The password used is taken directly from the `postgresql.auth.password` field in the `values.yaml` file.

**Use Case**: Ideal for local development, CI/CD testing, and ephemeral environments where simplicity is prioritized over security.

### Mode 2: Using an Existing Secret (Production Best Practice)

For production and other secure environments, the chart is designed to use a pre-existing Kubernetes Secret. This is the recommended approach as it completely decouples sensitive credentials from the Git repository.

**How it Works:**
1.  An administrator securely creates a Secret in the target namespace (e.g., `pedido-app-prod`) before deploying the application. This secret **must** contain a key named `password`.
    ```bash
    kubectl create secret generic production-v1-secret \
      --from-literal=password='a-very-strong-production-password' \
      -n pedido-app-prod
    ```
     
    It is also possible to define the secret using a private yaml file (you shouldn't share this file with anyone)
    
    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: production-v1-secret # The name of the secret
      namespace: pedido-app-prod # important, here you must specify the namespace in which your app lives
    type: Opaque
    stringData:
      password: VerySecureProdPassword # THe password that you want to use
    ```

    and then you can use `kubectl apply -f <secret-yaml-file.yaml>`
    
2.  In the environment-specific values file (e.g., `values-prod.yaml`), you specify the name of this secret:
    ```yaml
    # in values-prod.yaml
    postgresql:
      auth:
        existingSecret: "production-v1-secret"
    ```
3.  When Helm deploys the chart, both the PostgreSQL subchart and the backend deployment will be configured to source the database password from this single, secure, externally-managed secret. The chart itself will not create any secrets.




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
├── Chart.yaml
├── values.yaml
├── environments/
│   ├── dev/
│   │   ├── application-dev.yaml
│   │   └── values-dev.yaml
│   └── prod/
│       ├── application-prod.yaml
│       └── values-prod.yaml
├── templates/
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── backend-hpa.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── ingress-backend.yaml
│   ├── ingress-frontend.yaml
│   ├── configmap.yaml
│   └── secret.yaml
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

<a href="https://prod.estebanmf.space/" target="_blank" rel="noopener noreferrer">demo_production_page</a>


<a href="http://138.197.225.67/" target="_blank" rel="noopener noreferrer">demo_development_page</a>

## Installation Guide

This guide provides a step-by-step approach to deploying the application, starting from a quick test and progressing to a full, production-grade GitOps installation.

### Option 1: Quick Start / Test Deployment

This method is perfect for quickly launching the application in a new or temporary environment. It requires the least amount of configuration and deploys the application with its own bundled Ingress controller.

**Step 1: Install the Chart**
Run the following command to deploy the application into a new namespace called `pedido-app-test`.

```bash
helm install pedido-app-prod esteban-charts/parcial-1 \
  --namespace pedido-app-test \
  --create-namespace
```

**Step 2: Find the Application IP Address**
After a minute or two, the bundled Ingress controller will be assigned an external IP address by your cloud provider. You can find it by running:

```bash
kubectl get ingress -n pedido-app-test
# You will see an output like this:
# NAME                               CLASS   HOSTS   ADDRESS          PORTS   AGE
# pedido-app-prod-ingress-backend    nginx   *       143.244.201.58   80      2m33s
# pedido-app-prod-ingress-frontend   nginx   *       143.244.201.58   80      2m33s
```

You can now access your application directly using the IP address from the `ADDRESS` column (e.g., `http://143.244.201.58`).

---

### Option 2: Production-Like Manual Installation

This approach is for setting up a more robust environment manually. It involves creating a dedicated secret for the database and making a conscious choice about how to manage the Ingress controller, but still without using ArgoCD.

**Step 2.1: Create the Production Secret**
For a production environment, you must create a Kubernetes Secret to hold the database password externally. You have two options:

*   **Option A (Command-line):**
    ```bash
    kubectl create secret generic production-v1-secret \
      --from-literal=password='a-very-strong-production-password' \
      -n pedido-app-prod
    ```

*   **Option B (YAML file):** Create a file (e.g., `prod-secret.yaml`) and apply it. **Do not commit this file to Git.**
    ```yaml
    # prod-secret.yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: production-v1-secret
      namespace: pedido-app-prod
    type: Opaque
    stringData:
      password: VerySecureProdPassword
    ```
    Then apply it: `kubectl apply -f prod-secret.yaml`

**Step 2.2: Prepare Your Configuration File**
Create a local file named `my-prod-values.yaml`. This file will contain the bare minimum configuration needed. You must decide how to handle the Ingress controller:

*   **Using a Pre-existing Ingress Controller (Recommended):** If your cluster already has a shared NGINX Ingress controller, tell the chart not to install its own. This is the best practice.

    ```yaml
    # my-prod-values.yaml
    postgresql:
      auth:
        # Use the secret you created in the previous step
        existingSecret: "production-v1-secret"

    # Tell the chart to use the existing, cluster-wide Ingress controller
    ingress-nginx:
      enabled: false
    ```

*   **Using the Bundled Ingress Controller:** If you want this deployment to be self-contained with its own Ingress controller, you can omit the `ingress-nginx` block, as it defaults to `true`.

    ```yaml
    # my-prod-values.yaml
    postgresql:
      auth:
        # Use the secret you created in the previous step
        existingSecret: "production-v1-secret"
    # By omitting 'ingress-nginx', it will default to 'enabled: true'
    ```

**Step 2.3: Install the Chart**
Now, install the chart using your custom values file.

```bash
helm install pedido-app-prod esteban-charts/parcial-1 \
  -f my-prod-values.yaml \
  --namespace pedido-app-prod \
  --create-namespace
```
At this point, your application is running securely but is still only accessible via the Ingress controller's IP address.

**Step 2.4 (Optional): Upgrade to Use a Hostname**
Once the application is running, you can easily upgrade it to use a proper domain name.

```bash
helm upgrade pedido-app-prod esteban-charts/parcial-1 \
  --namespace pedido-app-prod \
  --set ingress.hosts.host="prod.estebanmf.space"
```
After running this, you must configure your DNS provider to point `prod.estebanmf.space` to your Ingress controller's external IP address, as described in the "DNS Configuration" section.

---

### Option 3: Full GitOps Deployment with ArgoCD (Recommended)

This is the most advanced and recommended method for managing deployments, as it automates the entire process from Git to your cluster.

**Step 1: Install and Access ArgoCD**
This is a one-time setup for your Kubernetes cluster. If ArgoCD is already installed, you can skip to the next step.

*   **Install ArgoCD:** Use the official manifest from the ArgoCD project. This will install all the necessary components in a new `argocd` namespace.
    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

*   **Access the ArgoCD UI:** To log in to the ArgoCD dashboard, you can expose it on your local machine using `port-forward`.
    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```
    You can now access the UI by navigating to `https://localhost:8080` in your browser.

*   **Retrieve the Initial Admin Password:** The initial password is automatically generated and stored in a secret. Retrieve it with the following command:
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```
    Log in to the UI with the username `admin` and the password you just retrieved.

**Step 2: Clone the Repository**
The easiest way to get started is to clone this repository, which contains the pre-configured ArgoCD Application definitions and environment-specific values files.

```bash
git clone https://github.com/EstebanForero/parcial-1.git
cd parcial-1
```

**Step 3: Prepare the Production Environment Secret**
Before deploying the production application, you must manually create the external secret in the `pedido-app-prod` namespace. ArgoCD will sync the application, but it relies on this secret already being present.

```bash
# Ensure the namespace exists first
kubectl create namespace pedido-app-prod

# Create the secret
kubectl create secret generic production-v1-secret \
  --from-literal=password='a-very-strong-production-password' \
  -n pedido-app-prod
```

**Step 4: Apply the ArgoCD Application Definitions**
These manifest files tell your ArgoCD instance which Git repository to monitor and where to deploy the applications. Apply them to your cluster:

```bash
# Apply the definition for the development environment
kubectl apply -f environments/dev/application-dev.yaml

# Apply the definition for the production environment
kubectl apply -f environments/prod/application-prod.yaml
```

ArgoCD will now take over. You will see `pedido-app-dev` and `pedido-app-prod` appear in the ArgoCD UI. Any changes you commit and push to the `development` or `master` branches of the repository will be automatically synchronized to your cluster.

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

## Installing the external Ingress Controller

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
- Backend: 1 replica, `50m CPU`/`256Mi memory` requests, `100m CPU`/`512Mi memory` limits.
- Frontend: 1 replica.
- PostgreSQL: 2Gi persistence. **Credentials provided via plaintext password in values file.**
- Ingress: `dev.parcial-1.local`

**Prod Environment (`values-prod.yaml`)**:
- Backend: 3 replicas, `100m CPU`/`100Mi memory` requests, `200m CPU`/`200Mi memory` limits.
- Frontend: 2 replicas.
- PostgreSQL: 10Gi persistence. **Credentials securely sourced from an existing secret named `production-v1-secret`.**
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
