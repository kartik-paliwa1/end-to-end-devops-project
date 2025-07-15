# End-to-End DevOps: A Cloud-Native Go Application on Kubernetes

This repository provides a comprehensive, end-to-end implementation of a modern cloud-native application stack. The project demonstrates the complete lifecycle of a Go-based application, from local development and containerization to a full production-grade deployment on a Kubernetes cluster. It serves as a prescriptive template for building, deploying, and managing robust, scalable, and observable applications.

The architecture emphasizes modern DevOps and GitOps principles, utilizing a curated, best-in-class toolchain. Key features include automated infrastructure provisioning, declarative application deployment with Argo CD, high-availability database management with the CloudNativePG operator, advanced traffic routing via the NGINX Gateway Fabric, and a comprehensive observability stack powered by Prometheus and Grafana.

-----

## Architecture and Technology Stack

The system architecture is designed for resilience, scalability, and maintainability within a Kubernetes environment. At its core is a simple Go application that interacts with a PostgreSQL database. This entire stack is deployed onto an Azure Kubernetes Service (AKS) cluster.

The logical flow of the system is as follows:

  * User traffic is directed to the **NGINX Gateway Fabric**, which serves as the entry point to the cluster and manages routing to backend services.
  * The **Go application**, containerized and deployed within Kubernetes, handles the core business logic.
  * The application's state is persisted in a high-availability **PostgreSQL cluster**, managed declaratively by the **CloudNativePG operator**. This operator handles tasks such as provisioning, failover, and backups.
  * The **Argo CD** platform continuously monitors the application's Git repository, ensuring that the live state of the Kubernetes deployment matches the desired state defined in the configuration files (GitOps).
  * The **kube-prometheus-stack** scrapes metrics from the application, Kubernetes components, and other services. These metrics are then visualized in **Grafana** dashboards, providing deep insight into system health and performance.
  * TLS certificates for all public-facing endpoints are automatically provisioned and managed by **cert-manager**, ensuring secure communication.

### Core Technologies and Roles

The project integrates a suite of powerful open-source tools, each chosen for a specific role in the application lifecycle. The following table provides a summary of the core technologies used.

| Tool                      | Role in Project                                     |
| ------------------------- | --------------------------------------------------- |
| Go (Golang)               | Application Programming Language                    |
| Docker                    | Local Containerization & Runtime Environment        |
| Kubernetes (AKS)          | Container Orchestration & Production Environment    |
| ksctl                     | Simplified Kubernetes Cluster Lifecycle Management  |
| bsf / ko                  | OCI Artifact / Go Application Build Tools           |
| PostgreSQL / CloudNativePG| Application Database (Managed via Kubernetes Operator)|
| Prometheus / Grafana      | Monitoring, Alerting, and Visualization (Observability) |
| NGINX Gateway Fabric      | Kubernetes Ingress and Traffic Management (Gateway API)|
| cert-manager              | Automated TLS Certificate Management                |
| Argo CD                   | GitOps Continuous Delivery                          |
| k6                        | Performance and Load Testing                        |

-----

## Prerequisites

Before proceeding, ensure the following tools are installed on your local machine and the necessary accounts are configured.

### Local Tools:

  * **git**: For cloning the repository.
  * **go**: The Go programming language toolchain.
  * **docker**: For building and running containers locally.
  * **kubectl**: The Kubernetes command-line tool.
  * **helm**: The package manager for Kubernetes.
  * **ksctl**: For simplified Kubernetes cluster management.
  * **k6**: For load testing.

### Accounts:

  * **Microsoft Azure Account**: An active subscription with permissions to create resource groups, AKS clusters, and related resources.
  * **Docker Hub Account**: An account on Docker Hub or another OCI-compliant container registry to store the built container images.

-----

## Part I: Local Development Environment Setup

This section guides you through running the application and its dependencies locally using Docker. This is ideal for quick functional testing and development.

### Clone the Repository

```bash
git clone https://github.com/kartik-paliwa1/end-to-end-devops-project.git
cd end-to-end-devops-project
```

### Build the Base and Application Images

This project uses `bsf` and `ko` to streamline the process of building OCI-compliant container images from Go source code without requiring a Dockerfile.

Initialize for the base image:

```bash
bsf init
```

Build and push the OCI artifact (replace with your Docker Hub credentials):

```bash
bsf oci pkgs --platform=linux/amd64 --tag=prod-v1 --push --dest-creds {DOCKERHUB_USERNAME}:{DOCKERHUB_PASSWORD}
```

Alternatively, build using `ko` (replace with your image repository name):

```bash
export KO_DOCKER_REPO={DOCKERHUB_USERNAME}/devops-project
ko build --bare -t v1.
```

### Launch Local Infrastructure

Run the required backing services (Grafana, Prometheus, PostgreSQL) as Docker containers.

Start Grafana:

```bash
docker run -d --name grafana -p 3000:3000 grafana/grafana
```

Start Prometheus:

```bash
docker run -d --name prometheus -p 9090:9090 -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

Start PostgreSQL:

```bash
docker run --name local-postgres -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword -e POSTGRES_DB=mydb -p 5432:5432 -d postgres
```

### Initialize Local Database Schema

Connect to the PostgreSQL container and create the necessary table for the application.

Connect to the database:

```bash
docker exec -it local-postgres psql -U myuser -d mydb
```

Inside the `psql` shell, run the following SQL command:

```sql
CREATE TABLE goals (
    id SERIAL PRIMARY KEY,
    goal_name TEXT NOT NULL
);
\q
```

### Run the Application Container

Run the main application container, ensuring it can connect to the local PostgreSQL database. Replace the image URL with the one you pushed to your registry. Note the use of `host.docker.internal` to allow the container to connect to services running on the Docker host.

```bash
docker run -d \
  --platform=linux/amd64 \
  -p 8080:8080 \
  -e DB_USERNAME=myuser \
  -e DB_PASSWORD=mypassword \
  -e DB_HOST=host.docker.internal \
  -e DB_PORT=5432 \
  -e DB_NAME=mydb \
  -e SSL=disable \
  {YOUR_DOCKERHUB_USERNAME}/devops-project:v1
```

-----

## Part II: Full Deployment to Kubernetes on Azure

This section details the complete process of provisioning a production-like environment on Azure Kubernetes Service (AKS) and deploying the entire application stack.

### 1\. Provision the AKS Cluster

Use `ksctl` to create a new AKS cluster.

```bash
ksctl create-cluster azure --name=application --version=1.29
```

### 2\. Configure kubectl Context

Switch your local `kubectl` context to point to the newly created cluster and export the Kubeconfig path.

```bash
ksctl switch-cluster --provider azure --region eastus --name devops-project
export KUBECONFIG="/Users/your-user/.ksctl/kubeconfig" # Adjust the path accordingly
```

### 3\. Install Core Cluster Services

Deploy the essential operators and services that form the foundation of the platform.

#### 3.1. Cert-Manager (for TLS)

Install cert-manager and configure it to work with the Gateway API.

```bash
# Install cert-manager components
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml

# Wait for pods to be ready
kubectl wait --for=condition=Available deployment --timeout=300s -n cert-manager --all

# Edit the deployment to enable Gateway API integration
kubectl edit deployment cert-manager -n cert-manager
# In the editor, find the 'args' section for the container and add:
# - --enable-gateway-api

# Restart the deployment to apply the changes
kubectl rollout restart deployment cert-manager -n cert-manager
```

#### 3.2. Kube-Prometheus-Stack (for Observability)

Install Prometheus and Grafana for monitoring using the community Helm chart.

```bash
# Add the Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install the kube-prometheus-stack in a dedicated namespace
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

#### 3.3. NGINX Gateway Fabric (for Ingress)

Install the NGINX Gateway Fabric, which implements the Kubernetes Gateway API.

```bash
# Install the Gateway API CRDs
kubectl kustomize "https://github.com/nginxinc/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.3.0" | kubectl apply -f -

# Install the NGINX Gateway Fabric using Helm
helm install ngf oci://ghcr.io/nginxinc/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway
```

#### 3.4. CloudNativePG Operator (for Database)

Install the CloudNativePG operator to manage PostgreSQL clusters within Kubernetes.

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.23/releases/cnpg-1.23.1.yaml
```

### 4\. Deploy and Configure the PostgreSQL Database

With the operator installed, create a resilient, three-instance PostgreSQL cluster.

#### 4.1. Create the Database Cluster

Apply the `Cluster` custom resource definition to provision the database.

```bash
cat << EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-postgresql
  namespace: default
spec:
  instances: 3
  storage:
    size: 1Gi
  bootstrap:
    initdb:
      database: goals_database
      owner: goals_user
      secret:
        name: my-postgresql-credentials
EOF
```

#### 4.2. Manage Database Credentials

Create the initial secret and then update the user's password within the database.

```bash
# Create the secret for the database user
kubectl create secret generic my-postgresql-credentials --from-literal=password='new_password' --from-literal=username='goals_user' --dry-run=client -o yaml | kubectl apply -f -

# Update the password inside the running PostgreSQL instance
kubectl exec -it my-postgresql-1 -- psql -U postgres -c "ALTER USER goals_user WITH PASSWORD 'new_password';"
```

#### 4.3. Initialize the Application Table

Port-forward to the primary database pod and create the `goals` table.

```bash
# Port-forward the primary PostgreSQL pod
kubectl port-forward my-postgresql-1 5432:5432 &

# Execute the CREATE TABLE statement
PGPASSWORD='new_password' psql -h 127.0.0.1 -U goals_user -d goals_database -c "
CREATE TABLE goals (
    id SERIAL PRIMARY KEY,
    goal_name VARCHAR(255) NOT NULL
);
"
# Kill the port-forward process
kill $!
```

### 5\. Deploy the Go Application

Deploy the main application to the cluster.

#### 5.1. Create Application Secrets

Create a Kubernetes secret that the application deployment will use to connect to the database. The username and password must be Base64 encoded.

```bash
# The values 'Z29hbHNfdXNlcg==' and 'bmV3X3Bhc3N3b3Jk' are the Base64 encodings of 'goals_user' and 'new_password' respectively.
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: postgresql-credentials
type: Opaque
data:
  password: bmV3X3Bhc3N3b3Jk
  username: Z29hbHNfdXNlcg==
EOF
```

#### 5.2. Apply Deployment Manifests

Apply the `deploy.yaml` file, which contains the Kubernetes Deployment, Service, and Gateway resources for the application.

```bash
kubectl apply -f deploy/deploy.yaml
```

### 6\. Install and Configure Argo CD for GitOps

Deploy Argo CD to manage application deployments declaratively.

#### 6.1. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### 6.2. Configure and Access Argo CD

For this demonstration, Argo CD is configured to run in insecure mode. **This is not recommended for production environments.**

```bash
# Patch the configmap to allow insecure server access
kubectl patch configmap argocd-cmd-params-cm -n argocd --patch '{"data":{"server.insecure":"true"}}'

# Restart the Argo CD server to apply the changes
kubectl rollout restart deployment argocd-server -n argocd

# Retrieve the initial admin password
kubectl get secret --namespace argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode ; echo
```

### 7\. Configure Network Routing via Gateway API

Expose the Argo CD dashboard to external traffic using a Gateway and HTTPRoute.

```bash
# Apply the HTTPRoute for Argo CD
kubectl apply -f route-argo.yaml

# Apply the ReferenceGrant to allow the Gateway in the nginx-gateway namespace
# to reference a Service in the argocd namespace.
kubectl apply -f referencegrant
```

-----

## Part III: Verification and Usage

After completing the deployment steps, verify that all services are accessible.

### Accessing the Grafana Dashboard

Get Admin Password:

```bash
kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Port-forward to the Grafana Service:

```bash
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
```

Access Grafana at `http://localhost:3000` and log in with the username `admin` and the retrieved password.

### Accessing the Argo CD and Application Dashboards

Get the External IP of the Gateway:

```bash
kubectl get gateway -n nginx-gateway
```

Look for the external IP address assigned to the `ngf` gateway.

  * **Access Argo CD**: Navigate to the external IP address in your browser. Log in with the username `admin` and the password retrieved in step 6.2.
  * **Access the Go Application**: The application will be available at the same external IP on a specific path defined in `deploy/deploy.yaml`.

-----

## Part IV: Performance and Load Testing

Use `k6` to run a simple load test against the deployed application endpoint.

### Update the Test Script

Open the `load.js` file and replace the placeholder URL with the actual external IP address of your application gateway.

### Execute the Load Test

```bash
k6 run load.js
```

Observe the output in your terminal and monitor the corresponding metrics in the Grafana dashboard to see how the system behaves under load.

-----

## Challenges Encountered & Key Considerations

Deploying a complex, multi-component system like this presents several challenges and highlights important architectural considerations.

1.  **Tooling Complexity and Abstraction**
    The project leverages high-level tools like `ksctl` for cluster creation and `ko` for building Go applications. While these tools dramatically simplify common workflows and reduce boilerplate configuration, they introduce their own layer of abstraction. When issues arise, debugging may require understanding the inner workings of these tools, not just the underlying technologies (like the Azure API or Docker image layers) they abstract away. This trade-off between convenience and control is a central theme in modern cloud-native development.

2.  **The Dev/Prod Parity Gap**
    A significant consideration is the inherent difference between the local Docker-based environment and the full Kubernetes deployment. The local setup uses a single, standalone PostgreSQL container, whereas the cloud environment uses a three-node, operator-managed, high-availability cluster via CloudNativePG. This discrepancy means that challenges related to stateful application behavior, network partitioning, leader election, and Kubernetes-native routing (via the Gateway API) cannot be replicated or tested locally. The local environment is suitable for rapid, inner-loop functional testing of the application code itself, but the Kubernetes environment is where integration, resilience, and operational issues must be validated.

3.  **Networking and Integration Complexity**
    The most intricate and error-prone aspects of this stack often reside at the integration points between components. A primary example is the interaction between the NGINX Gateway Fabric, cert-manager, and the application services. Correctly configuring the `Gateway`, `HTTPRoute`, and `ReferenceGrant` resources to allow secure, cross-namespace traffic routing is non-trivial. A `ReferenceGrant` is required to explicitly permit the Gateway in the `nginx-gateway` namespace to forward traffic to a Service in the `default` or `argocd` namespace. Debugging issues in this area often involves inspecting the status and events of multiple, interconnected Custom Resources (`Gateway`, `HTTPRoute`, `Certificate`, `Issuer`, `Service`) to trace the flow of configuration and identify the point of failure.

4.  **Security: From Tutorial to Production**
    This project's configuration includes several shortcuts for ease of setup that are **not suitable for a production environment**. It provides a foundational security posture but requires significant hardening.

      * **Insecure Secrets Handling**: The use of `kubectl create secret --from-literal` exposes sensitive data like passwords directly in the shell's history.
      * **Insecure Argo CD Access**: The command `kubectl patch configmap... --patch '{"data":{"server.insecure":"true"}}'` disables TLS for the Argo CD server, exposing login credentials and session data over the network.

    A "graduation path" from this tutorial to a production-ready deployment must include the following hardening steps:

    >   - **Secrets Management**: Integrate a dedicated secrets management solution like HashiCorp Vault, Azure Key Vault, or the Kubernetes External Secrets Operator to inject secrets into pods securely, avoiding storing them as plaintext in Kubernetes `Secret` objects.
    >   - **Secure Argo CD**: Remove the `server.insecure` patch. Instead, configure the NGINX Gateway to terminate TLS for the Argo CD route, using a valid certificate issued by cert-manager.
    >   - **Network Policies**: Implement Kubernetes `NetworkPolicy` resources to enforce a "zero-trust" networking model. Policies should be created to explicitly allow traffic from the application pods to the database pods while denying all other non-essential pod-to-pod communication.

-----

## Cleanup and Teardown

To avoid incurring ongoing cloud costs, destroy all the resources created during this project.

### Destroy Cloud Resources:

Use `ksctl` to delete the entire AKS cluster and its associated resources.

```bash
ksctl delete-cluster azure --name=application
```

### Destroy Local Resources:

Stop and remove all Docker containers and images created during the local setup.

```bash
# Stop all running containers
docker stop $(docker ps -a -q)

# Remove all containers
docker rm $(docker ps -a -q)

# Remove the container images (replace with your image names)
docker rmi {YOUR_DOCKERHUB_USERNAME}/devops-project:v1 grafana/grafana prom/prometheus postgres
```

-----

## Works Cited

kubesimplify/devops-project: This is the repo for DevOps ... - GitHub, accessed on July 15, 2025, [https://github.com/kubesimplify/devops-project](https://github.com/kubesimplify/devops-project)
