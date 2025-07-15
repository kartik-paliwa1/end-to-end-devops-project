# End-to-End DevOps Project

This repository provides a comprehensive, end-to-end implementation of a modern cloud-native application stack. 

-----

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

## 💻 Workflow 1: Local Development & Testing

This workflow allows you to run the entire application stack on your local machine using Docker.

### Step 1: Build OCI Artifacts

```bash
bsf init

bsf oci pkgs --platform=linux/amd64 --tag=prod-v1 --push --dest-creds {DOCKERHUB_USERNAME}:{DOCKERHUB_TOKEN}

KO_DOCKER_REPO={DOCKERHUB_USERNAME}/devops-project KO_DEFAULTBASEIMAGE={DOCKERHUB_USERNAME}/devops-proj:base ko build --bare -t v1 .
```

### Step 2: Run Dependencies and Application using Docker

```bash
docker run -d --name grafana -p 3000:3000 grafana/grafana

docker run -d --name prometheus -p 9090:9090 -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus

docker run --name local-postgres -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword -e POSTGRES_DB=mydb -p 5432:5432 -d postgres

docker exec -it local-postgres psql -U myuser -d mydb -c "CREATE TABLE goals (id SERIAL PRIMARY KEY, goal_name TEXT NOT NULL);"

docker run -d \
  --platform=linux/amd64 \
  -p 8080:8080 \
  -e DB_USERNAME=myuser \
  -e DB_PASSWORD=mypassword \
  -e DB_HOST=host.docker.internal \
  -e DB_PORT=5432 \
  -e DB_NAME=mydb \
  -e SSL=disable \
  {YOUR_APP_IMAGE_URL}
```

-----

## 🚀 Workflow 2: Full Deployment to AWS EKS

This workflow provisions cloud infrastructure and deploys the application stack to a managed Kubernetes cluster.

### Step 1: Environment Setup

```bash
git clone https://github.com/kartik-paliwa1/end-to-end-devops-project.git
cd end-to-end-devops-project
```

### Step 2: Configure Cloud Credentials

```bash
aws configure
ksctl cred
```

### Step 3: Create and Configure EKS Cluster

```bash
ksctl create cluster aws --name devops-project --region us-west-2 --version 1.29
aws eks update-kubeconfig --region us-west-2 --name devops-project-ksctl-managed
```

### Step 4: Install Core Kubernetes Components

#### NGINX Gateway Fabric

```bash
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v2.0.0/nginx-gateway.yaml
```

#### cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.15.1 \
  --set installCRDs=true
```

#### Kube Prometheus Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

#### CloudNativePG (PostgreSQL Operator)

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.23/releases/cnpg-1.23.1.yaml
```

#### Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch configmap argocd-cmd-params-cm -n argocd --patch '{"data":{"server.insecure":"true"}}'
kubectl rollout restart deployment argocd-server -n argocd
```

### Step 5: Configure `ClusterIssuer` for TLS Certificates

```bash
nano letsencrypt-prod-issuer.yaml
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-private-key
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
            - name: app-gateway
              namespace: default
```

```bash
kubectl apply -f letsencrypt-prod-issuer.yaml
```

### Step 6: Deploy and Configure the Database

```bash
kubectl apply -f pg_cluster.yaml
kubectl get pods --watch
kubectl port-forward svc/my-postgresql-rw 5432:5432
```

```bash
APP_PASSWORD=$(kubectl get secret my-postgresql-app -o jsonpath='{.data.password}' | base64 -d)

PGPASSWORD=$APP_PASSWORD psql -h 127.0.0.1 -U goals_user -d goals_database -c "
CREATE TABLE goals (
    id SERIAL PRIMARY KEY,
    goal_name VARCHAR(255) NOT NULL
);
"
```

### Step 7: Deploy the Application

```bash
kubectl apply -f deploy/deploy.yaml
```

### Step 8: Access Services

#### Argo CD

```bash
ARGO_PASSWORD=$(kubectl get secret --namespace argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode) && echo $ARGO_PASSWORD

kubectl port-forward svc/argocd-server -n argocd 8080:443

ssh -i /path/to/your/key.pem -L 8080:localhost:8080 user@your-server-ip
```

#### Grafana

```bash
GRAFANA_PASSWORD=$(kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode) && echo $GRAFANA_PASSWORD

kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80

ssh -i /path/to/your/key.pem -L 3000:localhost:3000 user@your-server-ip
```

### Step 9: Load Testing

```bash
k6 run load.js
```

-----

## ⚠️ Challenges faced while creating this project:

  * **Challenge 1: `bsf` Tooling Issues**

      * **Problem:** The `bsf` (BuildSafe) tool is no longer actively maintained and may not support newer Go versions or dependencies.
      * **Solution:** Transition to using **Chainguard Images** or a multi-stage `Dockerfile` to build minimal, secure container images for the Go application. This provides better security posture and long-term support.

  * **Challenge 2: NGINX Gateway Controller Failure**

      * **Problem:** The `Gateway` resource did not receive an external IP address. The `nginx-gateway-fabric` pods were in a `CrashLoopBackOff` state.
      * **Solution:** Completely uninstalled the broken deployment and reinstalled it using the single, official manifest (`kubectl apply -f https://.../nginx-gateway.yaml`), which ensures all CRDs are created correctly before the controller starts.

  * **Challenge 3: `cert-manager` Fails to Issue Certificates**

      * **Problem:** `kubectl get certificate` returned `No resources found` even though the `Gateway` was annotated correctly.
      * **Solution:** Uninstalled the broken Helm release and reinstalled a newer, compatible version of `cert-manager` (`v1.15.1` or higher) that has built-in support for the Gateway API.
