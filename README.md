# DSO Exam - GitOps Microservices Platform

A complete Event-Driven Microservices architecture deployed on Kubernetes (Kind) using GitOps principles with ArgoCD, Strimzi Kafka, and NGINX Ingress.

## 🏗 Architecture

The platform consists of three main application components and a robust infrastructure layer:

* **Frontend (Next.js):** Public-facing UI for submitting messages.
* **Backend (Go):** API that produces events to Kafka.
* **Consumer (Go):** Worker that consumes events and persists data to PostgreSQL.
* **Infrastructure:** PostgreSQL, Kafka (Strimzi Operator), NGINX Ingress, and Ops Tools.

### Data Flow
`User` -> `Ingress` -> `Frontend` -> `Backend` -> `Kafka Topic` -> `Consumer` -> `PostgreSQL`

---

## 🛠 Prerequisites

Ensure you have the following tools installed:
* [Docker](https://www.docker.com/)
* [Kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker)
* [Kubectl](https://kubernetes.io/docs/tasks/tools/)
* [Helm](https://helm.sh/)

---

## 🚀 Quick Start Guide

Follow these steps to spin up the entire environment from scratch.

### 1. Initialize the Cluster
Create the Kind cluster with the required port mappings (80/443 for Ingress).
Config file location : `kind/kind-config.yaml`

```bash
kind create cluster --config kind-config.yaml
```

Optional : export kubeconfig for using Len application

```bash
kind export kubeconfig --name devsecops-lab
```

### 2. Install ArgoCD
Deploy the GitOps engine.  
Config file location : `argocd/argocd-values.yaml`

```bash
# Add the ArgoCD Helm repo
helm repo add argo [https://argoproj.github.io/argo-helm](https://argoproj.github.io/argo-helm)
helm repo update

# Install ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 9.1.9 \
  --values argocd-values.yaml
```

### 3. Deploy Argo Application "control-center"
Argo application name `control-center` design for bootstrap infrastructure and manage application in a single kustomize.  
Config file location : `argocd/control-center/values.yaml`

```bash
kubectl apply -f values.yaml
```

---

## 🌐 Access Points

Once all pods are running, access the services locally via your browser.

| Service | URL | Credentials (If any) |
| :--- | :--- | :--- |
| **Main App** | [http://message.localhost](http://message.localhost) | - |
| **ArgoCD** | [http://argocd.localhost](http://argocd.localhost) | `admin` / *(Get password via CLI)* |
| **Kafka UI** | [http://kafka.localhost](http://kafka.localhost) | - |
| **PgAdmin** | [http://pgadmin.localhost](http://pgadmin.localhost) | `admin@admin.com` / `admin` |

> **Note:** If `*.localhost` domains do not resolve, add `127.0.0.1 message.localhost argocd.localhost` to your `/etc/hosts` file.

---

## 📂 Repository Structure

The repository follows a strict separation of concerns for GitOps, isolating infrastructure from application logic.
this example can be separate to multiple repository but for easy searching I merge it into one.
location path : `gitops/control-center`

```text
/
├── base/                               # Base application that need to have on every cluster
│   ├── app/                            # Base application list
│   │  ├── ingress-nginx.yaml           # App: Nginx Ingress
│   │  └── strimzi-operator.yaml        # App: strimzi-operator
│   ├── namespace/                      # Base namespace list
│   │  ├── backend.yaml                 # Namespace of backend
│   │  ├── consumer.yaml                # Namespace of consumer
│   │  ├── frontend.yaml                # Namespace of frontend
│   │  └── postgres.yaml                # Namespace of postgres
│   └── kustomization.yaml
└── overlay/                            # Overlay on each cluster
    └── lab-dev-th/                     # Can separate into multiple clusters, environments, regions
       ├── app/                         # Application list (in the future each directory can be repository)
       │  ├── backend/                  # App: backend
       │  │  ├── deployment.yaml
       │  │  ├── ingress.yaml
       │  │  ├── kustomization.yaml     # Manage apps, configs, secrets
       │  │  └── service.yaml
       │  ├── consumer/
       │  │  ├── deployment.yaml        # App: consumer
       │  │  ├── init-job.yaml          # Job: initial database
       │  │  ├── kustomization.yaml     # Manage apps, configs, secrets
       │  └── frontend/                 # App: frontend
       │     ├── deployment.yaml
       │     ├── ingress.yaml
       │     ├── kustomization.yaml     # Manage apps
       │     └── service.yaml
       ├── argocd/
       │  └── ingress.yaml              # Ingress for argoCD
       ├── kafka/                       # App: kafka
       │  ├── kafka-cluster.yaml
       │  ├── kafka-nodepool.yaml
       │  ├── kafka-ui.yaml             # App: kafka-ui
       │  └── topics.yaml
       ├── postgres/
       │  ├── pgadmin.yaml              # App: pgadmin
       │  └── postgres.yaml             # App: postgres
       └── kustomization.yaml           # Main kustomize manage cluster
```

---

## 🔧 Operational Cheatsheet

Useful commands for verifying and debugging the environment.

**1. Get ArgoCD Admin Password**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

**2.Check Kafka Broker Status**
```bash
kubectl get kafka -n kafka
```

**3. Verify Postgres Data Persistence Check if the messages are actually saved in the database:**
```bash
kubectl exec -it -n postgres deploy/postgres -- psql -U postgres -d messages -c "SELECT * FROM messages;"
```

**4. Debugging Pod Logs If something isn't working, check the logs for specific services:**
```bash
# Backend Logs
kubectl logs -n backend -l app=backend --tail=20

# Consumer Logs
kubectl logs -n consumer -l app=consumer --tail=20
```

---

## 🔮 Future Improvements

To elevate this platform to a production-grade enterprise standard, the following enhancements are proposed:

### 1. Advanced Security with HashiCorp Vault & External Secrets
**Current State:** Secrets (database credentials, API keys) are currently managed via Kustomize `secretGenerator` or Kubernetes native Secrets. While functional for development, this exposes secrets in the cluster state or requires manual management.

**Proposed Solution:** Implement a **"Secret Zero"** architecture using:
* **HashiCorp Vault:** To serve as the centralized, encrypted source of truth for all secrets.
* **External Secrets Operator (ESO):** To automatically fetch secrets from Vault and inject them into Kubernetes Native Secrets.

**Benefits:**
* **Rotation:** Automated rotation of database passwords without restarting pods.
* **Audit:** Centralized logging of who accessed which secret and when.
* **GitOps Compliance:** No sensitive data ever touches the Git repository, not even in encrypted formats (like Sealed Secrets).

### 2. Infrastructure as Code (IaC) with Terraform
**Current State:** The cluster bootstrapping (Kind creation) and initial tooling installation (ArgoCD via Helm) are currently performed using imperative shell scripts. This is prone to human error and difficult to replicate exactly across different environments (Dev, Staging, Prod).

**Proposed Solution:** adopt **Terraform** or **OpenTofu** to provision the entire bootstrapping layer.
* Use the **Kind Provider** to spin up the cluster configuration.
* Use the **Helm Provider** to install the initial ArgoCD instance in a declarative manner.

**Benefits:**
* **State Management:** Terraform keeps track of what is installed. If a configuration changes, Terraform only applies the "Delta" (difference) rather than re-running the whole script.
* **Idempotency:** Running the script 100 times results in the same stable state, preventing "configuration drift."
* **Disaster Recovery:** The entire cluster infrastructure can be destroyed and recreated with a single command (`terraform apply`).