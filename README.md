# 🗳️ Cloud-Native Voting App — Amazon EKS with CI/CD
 
A production-grade, distributed voting application deployed on **Amazon EKS** using **Kubernetes**, fully automated with a **GitHub Actions CI/CD pipeline**. The system runs on **spot instances** for cost efficiency, serves traffic over **HTTPS** with auto-renewing TLS certificates, and uses **NGINX Ingress** for intelligent traffic routing.
 
---
 
## 🏷️ Tech Stack
 
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.32-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![AWS EKS](https://img.shields.io/badge/AWS-EKS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Hub-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Python](https://img.shields.io/badge/Python-Flask-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-18-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![.NET](https://img.shields.io/badge/.NET-8-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-Alpine-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-3-0F1689?style=for-the-badge&logo=helm&logoColor=white)
![cert-manager](https://img.shields.io/badge/cert--manager-Let's%20Encrypt-003A70?style=for-the-badge)
 
---
 
## 📐 Architecture
 
```
User Browser
     │
     ▼ HTTPS
Route 53 (CNAME → ELB)
     │
     ▼
NGINX Ingress Controller        ← TLS termination via cert-manager
     │               │
     ▼               ▼
vote-service    result-service
     │               │
     ▼               │
  Redis ◄────── Worker (.NET) ──► PostgreSQL (db)
  :6379                              :5432
```
 
### Service Distribution
 
| Service | Language | Replicas | Role |
|---|---|---|---|
| **vote** | Python (Flask) | 2 | Accepts user votes → pushes to Redis |
| **worker** | .NET 8 | 1 | Reads from Redis → writes to PostgreSQL |
| **result** | Node.js | 2 | Reads from PostgreSQL → displays results |
| **redis** | Redis Alpine | 1 | In-memory vote queue |
| **postgres** | PostgreSQL 15 | 1 | Persistent vote storage |
 
### Data Flow
 
```
User votes → Vote App → Redis → .NET Worker → PostgreSQL → Result App → User sees results
              :80        :6379                  :5432           :80
```
 
---
 
## 📁 Project Structure
 
```
.
├── .github/
│   └── workflows/
│       └── ci-cd-pipeline.yml     # GitHub Actions CI/CD pipeline
├── k8s/
│   ├── vote-deployment.yaml       # Vote app — Deployment + Service
│   ├── result-deployment.yaml     # Result app — Deployment + Service
│   ├── worker-deployment.yaml     # Worker — Deployment
│   ├── redis-deployment.yaml      # Redis — Deployment + Service
│   ├── postgress-deployment.yaml  # PostgreSQL — Deployment + PVC + Service
│   ├── ingress.yaml               # NGINX Ingress with TLS + WebSocket support
│   ├── cert-cluster-issuer.yaml   # Let's Encrypt ClusterIssuer
│   └── db-credentials-secret.yaml # Kubernetes Secret for DB credentials
├── vote/
│   ├── app.py                     # Flask voting application
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── static/
│   └── templates/
├── result/
│   ├── server.js                  # Node.js results dashboard
│   ├── Dockerfile
│   ├── package.json
│   └── views/
└── worker/
    ├── Program.cs                 # .NET background worker
    ├── Worker.csproj
    └── Dockerfile
```
 
---
 
## ⚙️ Prerequisites
 
| Tool | Minimum version | Install |
|---|---|---|
| `kubectl` | latest | [kubernetes.io/docs/tasks/tools](https://kubernetes.io/docs/tasks/tools/) |
| `eksctl` | latest | [eksctl.io](https://eksctl.io/installation/) |
| `helm` | `>= 3.x` | [helm.sh](https://helm.sh/docs/intro/install/) |
| `aws` CLI | `>= 2.x` | [aws.amazon.com/cli](https://aws.amazon.com/cli/) |
| `docker` | latest | [docs.docker.com](https://docs.docker.com/get-docker/) |
 
### AWS account requirements
 
- IAM user with permissions for EC2, EKS, IAM, S3, Route 53
- AWS CLI configured: `aws configure`
- A domain registered in Route 53
---
 
## 🚀 Deployment Guide
 
### Phase 1 — Provision EKS cluster with spot instances
 
The cluster is provisioned using **spot instances** across 3 availability zones to significantly reduce compute costs while maintaining high availability.
 
```bash
eksctl create cluster -f spot-eks-lab-gerald.yaml
```
 
The cluster config (`spot-eks-lab-gerald.yaml`):
 
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: spot-eks-lab-gerald1
  region: us-east-1
  version: "1.32"
  tags:
    lab: spot-eks
    owner: gerald
 
availabilityZones:
  - us-east-1a
  - us-east-1b
  - us-east-1c
 
managedNodeGroups:
  - name: spot-nodes
    instanceTypes:
      - t3.small
      - t3a.small
      - t2.small
    spot: true
    minSize: 5
    maxSize: 5
    desiredCapacity: 5
    availabilityZones:
      - us-east-1a
      - us-east-1b
      - us-east-1c
    amiFamily: AmazonLinux2023
    labels:
      lifecycle: spot
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
```
 
Verify nodes are ready:
 
```bash
kubectl get nodes
```
 
---
 
### Phase 2 — Install EBS CSI driver
 
Required for PostgreSQL persistent volume provisioning.
 
```bash
# Associate OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster spot-eks-lab-gerald1 \
  --region us-east-1 \
  --approve
 
# Create IAM service account
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster spot-eks-lab-gerald1 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --region us-east-1
 
# Get role ARN and install addon
ROLE_ARN=$(aws iam get-role \
  --role-name eksctl-spot-eks-lab-gerald1-addon-iamservicea-Role1-Din5Ryb8Xp7o \
  --query 'Role.Arn' --output text)
 
aws eks create-addon \
  --cluster-name spot-eks-lab-gerald1 \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn $ROLE_ARN \
  --resolve-conflicts OVERWRITE \
  --region us-east-1
 
# Verify
kubectl get storageclass
kubectl get pods -n kube-system | grep ebs
```
 
---
 
### Phase 3 — Install NGINX Ingress Controller
 
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
 
helm install my-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
 
kubectl get pods -n ingress-nginx
```
 
---
 
### Phase 4 — Install cert-manager
 
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
 
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
 
kubectl get pods -n cert-manager
```
 
---
 
### Phase 5 — Apply Kubernetes manifests
 
```bash
# Create database credentials secret
kubectl apply -f k8s/db-credentials-secret.yaml
 
# Deploy all services
kubectl apply -f k8s/
 
# Verify all pods are running
kubectl get pods
kubectl get svc
kubectl get ingress
```
 
---
 
### Phase 6 — Configure DNS with Route 53
 
Get the Ingress ELB address:
 
```bash
kubectl get ingress voting-app-ingress \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```
 
Create CNAME records in Route 53 pointing `vote.geraldoti.com` and `result.geraldoti.com` to the ELB address.
 
---
 
### Phase 7 — Verify TLS certificate
 
```bash
kubectl get certificate -w
kubectl describe clusterissuer letsencrypt-prod
```
 
Wait for `READY = True` — cert-manager auto-renews every 60 days.
 
---
 
## 🔒 Security
 
### Kubernetes Secrets
 
Database credentials are stored as Kubernetes Secrets — never in plaintext in manifests:
 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: <your-secure-password>
  POSTGRES_DB: postgres
```
 
Referenced in deployments via `secretKeyRef` — credentials are injected at runtime, never hardcoded.
 
### HTTPS / TLS
 
All traffic is encrypted end-to-end:
 
```
Browser → HTTPS → Route 53 → ELB → NGINX Ingress (TLS termination) → Services (HTTP internally)
```
 
TLS certificates are issued by **Let's Encrypt** via cert-manager and renewed automatically.
 
### WebSocket support
 
The result app uses WebSockets for real-time vote updates. The Ingress is configured with extended timeouts and WebSocket support:
 
```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  nginx.ingress.kubernetes.io/websocket-services: "result-service"
```
 
---
 
## 🔄 CI/CD Pipeline
 
Every push to `main` automatically builds, pushes, and deploys the updated microservices to EKS.
 
```
git push → GitHub Actions → Docker Hub → EKS
```
 
### Pipeline steps
 
| Step | Action |
|---|---|
| Checkout | Clone repository |
| Docker login | Authenticate with Docker Hub |
| Build & push vote | `otigerald/voting-app:<git-sha>` |
| Build & push result | `otigerald/result-app:<git-sha>` |
| Build & push worker | `otigerald/worker-app:<git-sha>` |
| AWS credentials | Configure IAM access |
| Update kubeconfig | Connect to EKS cluster |
| Update manifests | Inject new image tags via `sed` |
| Deploy | `kubectl apply -f k8s/` |
| Verify rollout | Wait for all deployments to be healthy |
 
### Required GitHub secrets
 
| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `AWS_ACCESS_KEY_ID` | IAM access key |
| `AWS_SECRET_ACCESS_KEY` | IAM secret key |
 
---
 
## 🌐 Live Application
 
| Service | URL |
|---|---|
| **Vote** | [https://vote.geraldoti.com](https://vote.geraldoti.com) |
| **Results** | [https://result.geraldoti.com](https://result.geraldoti.com) |
 
---
 
## 💰 Cost Optimisation
 
The cluster uses **AWS spot instances** across 3 instance type families (`t3.small`, `t3a.small`, `t2.small`) and 3 availability zones. This provides:
 
- Up to **70% cost reduction** compared to on-demand instances
- **High availability** — if one spot instance is reclaimed, workloads reschedule across remaining nodes
- **Flexible capacity** — multiple instance types increases the pool of available spot capacity
---
 
## ⚠️ Known Limitations & Areas for Improvement
 
### 1. Database single point of failure
 
The current setup runs a single PostgreSQL instance. If it fails, the entire application is broken — votes cannot be written or read.
 
**Recommended improvement:** Implement a **primary/secondary PostgreSQL architecture**:
 
- **Primary** — accepts all write operations (votes from the worker)
- **Secondary (read replica)** — serves all read operations (result app queries)
- Automatic failover — if primary fails, secondary is promoted
This can be achieved using:
- **Bitnami PostgreSQL Helm chart** with `replication.enabled=true`
- **CloudNativePG** operator for Kubernetes-native PostgreSQL HA
- **AWS RDS Multi-AZ** — fully managed primary/secondary with automatic failover
### 2. Spot instance interruption handling
 
Spot instances can be reclaimed with 2 minutes notice. Stateful workloads (PostgreSQL) should not run on spot nodes. Recommended fix: add a dedicated on-demand node group for stateful services.
 
### 3. No horizontal pod autoscaling
 
The vote and result deployments have fixed replica counts. Under high traffic, pods will not scale. Recommended fix: add HorizontalPodAutoscaler resources targeting CPU/memory thresholds.
 
### 4. Secrets management
 
Database credentials are stored in Kubernetes Secrets (base64 encoded, not encrypted at rest by default). Recommended fix: use **AWS Secrets Manager** with the External Secrets Operator, or enable EKS envelope encryption for Secrets.
 
---
 
## 👤 Author
 
**Gerald Oti** — DevOps / SRE Engineer
 
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Gerald%20Oti-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/geraldoti)
[![GitHub](https://img.shields.io/badge/GitHub-goti13-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/goti13)
 
---
 
## 📄 License
 
Infrastructure and deployment code authored by Gerald Oti. Application source code based on [example-voting-app](https://github.com/dockersamples/example-voting-app) by Docker Samples, licensed under the Apache 2.0 License.
 