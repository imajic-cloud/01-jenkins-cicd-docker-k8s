# Jenkins CI/CD Pipeline with Docker, Helm, and Kubernetes

A production-ready CI/CD pipeline implementation using Jenkins, Docker, Helm, and Kubernetes. This project demonstrates automated build, test, and deployment workflows using industry-standard DevOps tools and practices.

## Overview

This project showcases a complete CI/CD pipeline where Jenkins orchestrates the entire application lifecycle - from source code commit to production deployment on Kubernetes. The pipeline runs on a Linux agent, builds containerized applications, and deploys them using Helm charts.

## Architecture

```
GitHub → Jenkins (Linux Agent) → Docker Build → Docker Hub → Helm Deploy → Kubernetes Cluster
```

**Pipeline Flow:**
1. Developer pushes code to GitHub repository
2. Jenkins detects changes via webhook or polling
3. Pipeline executes on Linux agent (WSL)
4. Docker image is built from source code
5. Image is tagged and pushed to Docker Hub
6. Helm chart deploys the application to Kubernetes
7. Application runs on Kubernetes cluster (Docker Desktop)

## Technologies Stack

| Category | Technology |
|----------|-----------|
| **CI/CD Platform** | Jenkins |
| **Jenkins Architecture** | Windows Controller + Linux Agent (WSL) |
| **Container Runtime** | Docker |
| **Container Registry** | Docker Hub |
| **Orchestration** | Kubernetes (Docker Desktop) |
| **Package Manager** | Helm 3 |
| **Version Control** | GitHub |
| **Operating System** | Linux (WSL for agent) |

## Project Structure

```
01-jenkins-cicd-docker-k8s/
│
├── Jenkinsfile                 # Pipeline definition
├── Dockerfile                  # Container image specification
├── app/                        # Application source code
│   ├── src/
│   └── package.json
│
├── helm/                       # Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
│
├── k8s/                        # Kubernetes manifests (alternative)
│   ├── deployment.yaml
│   └── service.yaml
│
└── README.md
```

## Prerequisites

Before running this project, ensure you have:

- **Jenkins** (v2.400+) - Installed and running
- **Docker** (v20.x or higher) - For building images
- **Docker Desktop** - With Kubernetes enabled
- **Helm** (v3.x) - For Kubernetes deployments
- **WSL2** (Windows Subsystem for Linux) - For Linux agent
- **kubectl** - Configured to connect to your cluster
- **GitHub Account** - For source code repository
- **Docker Hub Account** - For image registry

## Setup Instructions

### 1. Jenkins Configuration

#### Install Jenkins

**On Windows:**
```powershell
# Download Jenkins installer from jenkins.io
# Or use Chocolatey:
choco install jenkins
```

**Access Jenkins:**
- Navigate to `http://localhost:8080`
- Complete initial setup wizard
- Install suggested plugins

#### Configure Linux Agent (WSL)

1. **Install WSL2 on Windows:**
```powershell
wsl --install
wsl --set-default-version 2
```

2. **Install Ubuntu in WSL:**
```bash
wsl --install -d Ubuntu-22.04
```

3. **Install Java on WSL (required for Jenkins agent):**
```bash
sudo apt update
sudo apt install openjdk-11-jdk -y
java -version
```

4. **Install Docker in WSL:**
```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Start Docker service
sudo service docker start
```

5. **Configure Jenkins Agent:**
   - In Jenkins: Manage Jenkins → Nodes → New Node
   - Node name: `linux-agent`
   - Type: Permanent Agent
   - Remote root directory: `/home/<username>/jenkins`
   - Labels: `linux`
   - Launch method: Launch agent via SSH
   - Host: `localhost` (or WSL IP)
   - Credentials: Add SSH credentials for WSL user

### 2. Docker Hub Credentials

Add Docker Hub credentials to Jenkins:

1. Go to: Manage Jenkins → Credentials → Global
2. Click "Add Credentials"
3. Kind: Username with password
4. ID: `dockerhub-credentials`
5. Username: Your Docker Hub username
6. Password: Your Docker Hub password or access token

### 3. GitHub Credentials

Add GitHub credentials to Jenkins:

1. Go to: Manage Jenkins → Credentials → Global
2. Click "Add Credentials"
3. Kind: Username with password (or SSH key)
4. ID: `github-credentials`
5. Username: Your GitHub username
6. Password: GitHub Personal Access Token

### 4. Kubernetes Configuration

Copy kubeconfig to Jenkins agent:

```bash
# On WSL (Linux agent)
mkdir -p ~/.kube

# Copy kubeconfig from Windows to WSL
cp /mnt/c/Users/<YourUsername>/.kube/config ~/.kube/config

# Verify connection
kubectl get nodes
```

### 5. Install Required Jenkins Plugins

Install these plugins from Manage Jenkins → Plugins:

- **Docker Pipeline** - Docker build and push
- **Kubernetes CLI** - kubectl commands
- **Git** - GitHub integration
- **Pipeline** - Pipeline support
- **Credentials Binding** - Secure credential handling
- **Blue Ocean** (optional) - Modern UI

## Jenkins Pipeline

### Jenkinsfile

```groovy
pipeline {
    agent {
        label 'linux'  // Run on Linux agent
    }
    
    environment {
        DOCKER_IMAGE = "yourusername/myapp"
        DOCKER_TAG = "${BUILD_NUMBER}"
        DOCKER_CREDENTIALS = credentials('dockerhub-credentials')
        KUBECONFIG = "${HOME}/.kube/config"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                git branch: 'main',
                    credentialsId: 'github-credentials',
                    url: 'https://github.com/yourusername/your-repo.git'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh """
                    docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                    docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                """
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing image to Docker Hub...'
                sh """
                    echo ${DOCKER_CREDENTIALS_PSW} | docker login -u ${DOCKER_CREDENTIALS_USR} --password-stdin
                    docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                    docker push ${DOCKER_IMAGE}:latest
                """
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying to Kubernetes using Helm...'
                sh """
                    helm upgrade --install myapp ./helm \
                        --set image.repository=${DOCKER_IMAGE} \
                        --set image.tag=${DOCKER_TAG} \
                        --namespace default
                """
            }
        }
        
        stage('Verify Deployment') {
            steps {
                echo 'Verifying deployment...'
                sh """
                    kubectl rollout status deployment/myapp
                    kubectl get pods -l app=myapp
                    kubectl get svc myapp
                """
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully! ✅'
        }
        failure {
            echo 'Pipeline failed! ❌'
        }
        always {
            echo 'Cleaning up...'
            sh 'docker logout'
        }
    }
}
```

### Pipeline Stages Explained

1. **Checkout** - Clones the latest code from GitHub
2. **Build Docker Image** - Creates container image with build number tag
3. **Push to Docker Hub** - Uploads image to registry
4. **Deploy to Kubernetes** - Uses Helm to deploy/upgrade application
5. **Verify Deployment** - Confirms pods are running successfully

## Running the Pipeline

### Method 1: Manual Trigger

1. Open Jenkins dashboard
2. Click on your pipeline job
3. Click "Build Now"
4. Monitor the build in "Build History"
5. View console output for details

### Method 2: GitHub Webhook (Automatic)

1. **In GitHub repository:**
   - Go to Settings → Webhooks → Add webhook
   - Payload URL: `http://your-jenkins-url:8080/github-webhook/`
   - Content type: `application/json`
   - Events: "Just the push event"

2. **In Jenkins job:**
   - Configure → Build Triggers
   - Check "GitHub hook trigger for GITScm polling"

Now every push to GitHub will trigger the pipeline automatically!

### Method 3: Polling (Alternative)

In Jenkins job configuration:
- Build Triggers → Poll SCM
- Schedule: `H/5 * * * *` (checks every 5 minutes)

## Kubernetes Deployment

### View Deployed Application

```bash
# Get all resources
kubectl get all

# Get pods
kubectl get pods -l app=myapp

# Get service
kubectl get svc myapp

# Get deployment details
kubectl describe deployment myapp

# View logs
kubectl logs -f deployment/myapp
```

### Access the Application

```bash
# Get service URL (if LoadBalancer)
kubectl get svc myapp

# Port forward for testing
kubectl port-forward svc/myapp 8080:80

# Access at: http://localhost:8080
```

### Scale the Application

```bash
# Scale to 3 replicas
kubectl scale deployment myapp --replicas=3

# Verify scaling
kubectl get pods -l app=myapp
```

## Helm Chart Configuration

### values.yaml

```yaml
replicaCount: 2

image:
  repository: yourusername/myapp
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80
  targetPort: 8080

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### Helm Commands

```bash
# Install chart
helm install myapp ./helm

# Upgrade deployment
helm upgrade myapp ./helm --set image.tag=v2.0

# List releases
helm list

# Rollback to previous version
helm rollback myapp

# Uninstall
helm uninstall myapp
```

## Pipeline Features

✅ **Linux-Based Execution** - All commands run on Linux agent for consistency  
✅ **Docker Image Automation** - Automated build, tag, and push  
✅ **Kubernetes Deployment** - Helm-based deployments with version control  
✅ **Credential Management** - Secure handling of Docker Hub and GitHub credentials  
✅ **Automated Testing** - Can integrate unit tests, security scans  
✅ **Deployment Verification** - Automatic health checks post-deployment  
✅ **Rollback Capability** - Helm enables easy rollback to previous versions  

## Monitoring & Verification

### Jenkins Build Status

- **Blue Ocean UI**: Modern pipeline visualization
- **Console Output**: Detailed logs for each stage
- **Build History**: Track success/failure trends

### Kubernetes Health Checks

```bash
# Check pod health
kubectl get pods --watch

# View events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check resource usage
kubectl top pods
kubectl top nodes

# Describe pod for issues
kubectl describe pod <pod-name>
```

## Troubleshooting

### Issue: Jenkins agent won't connect

**Solution:**
```bash
# Check SSH connection from Jenkins to WSL
ssh username@localhost

# Verify Java is installed on agent
java -version

# Check Jenkins agent logs
tail -f /home/username/jenkins/logs/agent.log
```

### Issue: Docker build fails

**Solution:**
```bash
# Verify Docker is running on agent
docker ps

# Check Dockerfile syntax
docker build -t test .

# View build logs
docker build --no-cache -t test . 2>&1 | tee build.log
```

### Issue: Cannot push to Docker Hub

**Solution:**
```bash
# Verify credentials
docker login

# Check image exists
docker images | grep myapp

# Manual push test
docker push yourusername/myapp:latest
```

### Issue: Helm deployment fails

**Solution:**
```bash
# Verify Helm is installed
helm version

# Check kubeconfig
kubectl config view

# Test Helm chart
helm install --dry-run --debug myapp ./helm

# View Helm release status
helm status myapp
```

### Issue: Pods not starting

**Solution:**
```bash
# Check pod status
kubectl get pods -l app=myapp

# View pod logs
kubectl logs <pod-name>

# Describe pod for events
kubectl describe pod <pod-name>

# Check image pull
kubectl get events | grep -i pull
```

### Issue: "Permission denied" on WSL

**Solution:**
```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Restart WSL
wsl --shutdown
wsl

# Verify
docker ps
```

## Best Practices Implemented

- ✅ **Agent Isolation** - Builds run on dedicated Linux agent, not controller
- ✅ **Credentials Security** - No hardcoded secrets in Jenkinsfile
- ✅ **Idempotent Deployments** - Helm upgrade creates or updates
- ✅ **Tagging Strategy** - Images tagged with build numbers for traceability
- ✅ **Health Checks** - Kubernetes liveness and readiness probes
- ✅ **Resource Limits** - Prevents runaway containers
- ✅ **Clean Workspace** - Pipeline cleans up after execution

## Security Considerations

- Jenkins credentials stored securely
- Docker Hub access tokens instead of passwords
- Kubernetes RBAC for access control
- Container image scanning (can be added)
- Secret management with Kubernetes Secrets
- WSL agent isolated from Windows host

## Performance Optimizations

- Docker layer caching for faster builds
- Multi-stage Dockerfiles to reduce image size
- Kubernetes resource limits and requests
- Horizontal Pod Autoscaler for traffic spikes
- Helm chart values for environment-specific configs

## Future Enhancements

- [ ] Add automated testing stage (unit, integration)
- [ ] Implement security scanning (Trivy, Snyk)
- [ ] Separate CI and CD pipelines
- [ ] Add staging environment deployment
- [ ] Migrate Linux agent to AWS EC2
- [ ] Implement blue-green deployment strategy
- [ ] Add Slack/email notifications
- [ ] Integrate Prometheus monitoring
- [ ] Add SonarQube code quality checks
- [ ] Implement GitOps with ArgoCD

## Key Learnings

This project demonstrates:

- **Jenkins Architecture**: Understanding controller vs agent separation
- **Linux vs Windows**: Why Linux agents are preferred for DevOps pipelines
- **Container Workflows**: Complete Docker build, push, deploy cycle
- **Kubernetes Operations**: Deployment, scaling, and management
- **Helm Benefits**: Version control and rollback capabilities
- **CI/CD Best Practices**: Automated, repeatable, reliable deployments
- **Troubleshooting Skills**: Debugging pipeline, Docker, and Kubernetes issues

## Testing the Pipeline

### End-to-End Test

1. **Make a code change:**
```bash
# Edit application code
vim app/src/index.js

# Commit and push
git add .
git commit -m "Update application"
git push origin main
```

2. **Watch pipeline execute:**
   - Jenkins automatically detects change
   - Pipeline runs through all stages
   - New Docker image is built and pushed
   - Kubernetes deployment updates

3. **Verify deployment:**
```bash
# Check new pods rolling out
kubectl get pods -w

# Verify new version
kubectl describe pod <pod-name> | grep Image
```

## Commands Reference

### Jenkins CLI

```bash
# Trigger build remotely
curl -X POST http://localhost:8080/job/myapp-pipeline/build \
  --user username:api-token

# Get build status
java -jar jenkins-cli.jar -s http://localhost:8080/ get-job myapp-pipeline
```

### Docker Commands

```bash
# Build image
docker build -t myapp:latest .

# Push to registry
docker push yourusername/myapp:latest

# Run locally
docker run -p 8080:80 myapp:latest

# Clean up unused images
docker image prune -a
```

### Kubernetes Commands

```bash
# Apply manifests
kubectl apply -f k8s/

# Delete deployment
kubectl delete deployment myapp

# Get logs from all pods
kubectl logs -l app=myapp --tail=100

# Execute command in pod
kubectl exec -it <pod-name> -- /bin/sh
```

## Resources

- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Docker Documentation](https://docs.docker.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)

## Project Status

✅ **Pipeline Status**: Fully operational and green  
✅ **Docker Build**: Successfully builds and pushes images  
✅ **Kubernetes Deployment**: Helm deployments working perfectly  
✅ **Automation**: Full CI/CD automation achieved  

## Author

**DevOps Portfolio Project**

This project showcases hands-on experience with:
- Jenkins CI/CD pipeline design and implementation
- Docker containerization and registry management
- Kubernetes orchestration and deployment
- Helm package management
- Linux system administration
- DevOps automation and best practices

---

## License

This project is open source and available for educational purposes.

## Contributing

Feel free to fork this project and submit pull requests with improvements!

---

**⭐ If you find this project helpful, please give it a star