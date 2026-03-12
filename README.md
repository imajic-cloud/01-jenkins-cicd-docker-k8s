# Jenkins CI/CD Pipeline: Docker → Helm → Kubernetes Deployment

![Jenkins](https://img.shields.io/badge/CI-Jenkins-red)
![Docker](https://img.shields.io/badge/container-Docker-blue)
![Kubernetes](https://img.shields.io/badge/orchestrator-Kubernetes-blue)
![Helm](https://img.shields.io/badge/package-Helm-purple)
![License](https://img.shields.io/badge/license-MIT-green)

A hands-on **DevOps portfolio project** demonstrating a complete CI/CD pipeline using **Jenkins, Docker, Helm, and Kubernetes**, running locally on a Windows machine with **WSL2**.

---

# Architecture

### Pipeline Flow

```
Developer Push
      │
      ▼
GitHub Repository
      │
      ▼
Jenkins Controller (Windows)
      │
      │ SSH
      ▼
Linux Agent (WSL2 Ubuntu)
      │
      ▼
Docker Build
      │
      ▼
Docker Hub
      │
      ▼
Helm Deployment
      │
      ▼
Kubernetes Cluster (Docker Desktop)
      │
      ▼
Running Pods
```

---

# Tech Stack

| Category             | Technology                              |
| -------------------- | --------------------------------------- |
| CI/CD Platform       | Jenkins                                 |
| Jenkins Architecture | Windows Controller + Linux Agent (WSL2) |
| Containerization     | Docker (nginx:alpine)                   |
| Container Registry   | Docker Hub                              |
| Kubernetes           | Docker Desktop (local, Kubeadm)         |
| Package Manager      | Helm 3                                  |
| Version Control      | GitHub                                  |
| OS (Agent)           | Ubuntu 24.04 (WSL2)                     |

---

# Project Structure

```
01-jenkins-cicd-docker-k8s/
├── Jenkinsfile              # 5-stage pipeline definition
├── Dockerfile               # nginx:alpine container image
├── helm/                    # Helm chart
│   ├── Chart.yaml           # Chart metadata (v0.1.0)
│   ├── values.yaml          # Default values and feature flags
│   ├── charts/              # Chart dependencies
│   └── templates/
│       ├── _helpers.tpl         # Template helper functions
│       ├── NOTES.txt            # Post-install notes
│       ├── deployment.yaml      # RollingUpdate, liveness/readiness probes
│       ├── service.yaml         # NodePort service on port 80
│       ├── serviceaccount.yaml  # Pod identity
│       ├── ingress.yaml         # External routing (disabled by default)
│       ├── hpa.yaml             # Horizontal Pod Autoscaler (disabled by default)
│       ├── httproute.yaml       # Gateway API route (disabled by default)
│       └── tests/               # Helm test hooks
└── README.md
```

---

# CI/CD Pipeline

The Jenkins pipeline runs **entirely on the Linux agent (WSL2)**.

```groovy
agent { label 'linux' }
```

Pipeline stages:

| Stage                | Description                                    |
| -------------------- | ---------------------------------------------- |
| Checkout             | Clone repository from GitHub                   |
| Build Docker Image   | Build image tagged with `build-${BUILD_NUMBER}`|
| Login Docker Hub     | Authenticate using Jenkins credentials         |
| Push Docker Image    | Push versioned image to Docker Hub             |
| Deploy to Kubernetes | Helm upgrade/install deployment                |

---

# Docker Build

The Dockerfile builds a lightweight container using **nginx:alpine**.

Images are tagged with the Jenkins build number:

```
ikomajic/project2-demo:build-${BUILD_NUMBER}
```

---

# Helm Deployment

Helm manages Kubernetes resources and handles upgrades.

Deploy command used by Jenkins:

```bash
helm upgrade --install project2 helm \
  --set image.repository=ikomajic/project2-demo \
  --set image.tag=build-${BUILD_NUMBER}
```

Helm ensures **idempotent deployments** — first run installs, later runs upgrade.

---

# Kubernetes Resources

The Helm chart creates the following resources:

| Resource       | Status          | Purpose                              |
| -------------- | --------------- | ------------------------------------ |
| Deployment     | ✅ Always on    | Runs containerized application       |
| Service        | ✅ Always on    | Exposes application via NodePort     |
| ServiceAccount | ✅ Always on    | Pod identity for Kubernetes RBAC     |
| Ingress        | ⚙️ Optional     | External routing (enable in values)  |
| HPA            | ⚙️ Optional     | Horizontal Pod Autoscaling           |
| HTTPRoute      | ⚙️ Optional     | Gateway API routing                  |

To enable optional resources, update `values.yaml`:

```yaml
ingress:
  enabled: true

autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 10

httpRoute:
  enabled: true
```

Deployment configuration includes:

- RollingUpdate strategy (maxUnavailable: 0, maxSurge: 1)
- Liveness probe on `/`
- Readiness probe on `/`
- Configurable replica count

---

# Setup Instructions

## 1. Enable Kubernetes in Docker Desktop

Docker Desktop → Settings → Kubernetes → Enable Kubernetes (Kubeadm) → Apply & Restart

Verify cluster:

```bash
kubectl config use-context docker-desktop
kubectl get nodes
# NAME             STATUS   ROLES           VERSION
# docker-desktop   Ready    control-plane   v1.34.1
```

---

## 2. Install Jenkins

Run PowerShell as Administrator:

```powershell
choco install jenkins -y
```

Access Jenkins at `http://localhost:8080` and complete setup wizard.

---

## 3. Configure Linux Agent (WSL2)

Install SSH server:

```bash
sudo apt install openssh-server -y
sudo service ssh start
```

Add node in Jenkins — **Manage Jenkins → Nodes → New Node**:

| Field                     | Value                               |
| ------------------------- | ----------------------------------- |
| Node Name                 | linux-agent                         |
| Labels                    | linux                               |
| Remote Directory          | /home/username/jenkins              |
| Launch Method             | Launch agent via SSH                |
| Host                      | localhost                           |
| Host Key Verification     | Non verifying Verification Strategy |

---

## 4. Configure Git in Jenkins

**Manage Jenkins → Tools → Git**

Set executable path to: `git`

---

## 5. Add Docker Hub Credentials

**Manage Jenkins → Credentials → Global → Add Credentials**

| Field    | Value                     |
| -------- | ------------------------- |
| Kind     | Username with password    |
| ID       | dockerhub                 |
| Username | Docker Hub username       |
| Password | Docker Hub password/token |

---

## 6. Create Jenkins Pipeline Job

**New Item → Pipeline → Pipeline Script from SCM**

| Field       | Value                                                       |
| ----------- | ----------------------------------------------------------- |
| SCM         | Git                                                         |
| Repository  | https://github.com/imajic-cloud/01-jenkins-cicd-docker-k8s |
| Branch      | */main                                                      |
| Script Path | Jenkinsfile                                                 |

---

# Running the Pipeline

1. Open Jenkins dashboard
2. Open pipeline job
3. Click **Build Now**
4. Watch all 5 stages go green ✅

---

# Verify Deployment

```bash
# Check pods are running
kubectl get pods
# project2-helm-xxxx   1/1   Running   0   1m

# Check service
kubectl get svc
# project2-helm   NodePort   10.96.x.x   80:3XXXX/TCP

# Access the app
kubectl port-forward svc/project2-helm 8090:80
```

Open browser: `http://localhost:8090` → nginx welcome page ✅

---

# Key DevOps Concepts Demonstrated

### Jenkins Controller vs Agent
The Jenkins controller manages scheduling and UI, while the Linux agent (WSL2) executes build tasks. This isolates builds and prevents resource exhaustion on the Jenkins server.

### Pipeline as Code
The Jenkinsfile lives in the repository, enabling version-controlled CI/CD pipelines.

### Containerized Deployment
Applications are packaged into Docker images, ensuring consistent environments across machines.

### Helm Idempotent Deployments
`helm upgrade --install` supports both installation and upgrade in one command, making pipelines safe to re-run.

### Secure Credentials
Docker Hub credentials are stored in Jenkins and injected securely at runtime via `withCredentials`.

### Zero-Downtime Deployment
RollingUpdate strategy ensures pods are replaced gradually with no downtime.

### Production-Ready Helm Chart
Optional resources (Ingress, HPA, HTTPRoute) are included and can be enabled via feature flags in `values.yaml` — ready for production without modifying templates.

---

# Troubleshooting

**Jenkins agent offline**
```bash
sudo service ssh status
ssh username@localhost
```
Set Host Key Verification to "Non verifying Verification Strategy".

**Git clone fails (CreateProcess error)**

Set Git path in Jenkins Tools to just `git`.

**Application not reachable**
```bash
kubectl port-forward svc/project2-helm 8090:80
```

**Docker build fails**
```bash
docker ps
sudo usermod -aG docker $USER
```

---

# Project Comparison

|                | Project 1 (this repo)  | Project 2      |
| -------------- | ---------------------- | -------------- |
| CI/CD          | Jenkins                | GitHub Actions |
| Kubernetes     | Docker Desktop (local) | AWS EKS        |
| Infrastructure | Local                  | Cloud          |
| Registry       | Docker Hub             | AWS ECR        |
| Deployment     | Helm                   | Terraform      |

---

# Author

**Ivan Majic** — DevOps Portfolio Project

Demonstrates Jenkins CI/CD pipelines, Docker containerization, Helm-based Kubernetes deployments, and local DevOps infrastructure setup with WSL2.