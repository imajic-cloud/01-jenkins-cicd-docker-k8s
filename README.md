# Project2 – CI/CD Pipeline with Jenkins, Docker, Helm, and Kubernetes

## Overview
This project demonstrates a complete CI/CD pipeline built with Jenkins that runs on a Linux agent. The pipeline builds a Docker image, pushes it to Docker Hub, and deploys the application to Kubernetes using Helm.

The focus of this project is to learn real-world DevOps practices using Linux-based tooling and understand CI/CD concepts clearly.

---

## Tools & Technologies
- Jenkins (Windows controller, Linux agent via WSL)
- Docker
- Docker Hub
- Kubernetes (Docker Desktop)
- Helm
- GitHub

---

## CI/CD Pipeline Flow
1. Jenkins runs the pipeline on a Linux agent
2. Source code is cloned from GitHub
3. Docker image is built
4. Docker image is pushed to Docker Hub
5. Application is deployed to Kubernetes using Helm

---

## Jenkins Pipeline
- Pipeline is executed only on a Linux agent
- Git checkout is done explicitly inside the pipeline
- Jenkins credentials are used for GitHub and Docker Hub
- Shell (`sh`) commands are used instead of Windows-specific commands

---

## Kubernetes Deployment
- Kubernetes cluster is provided by Docker Desktop
- The Linux agent uses kubeconfig to connect to the cluster
- Deployment is managed using Helm

---

## Key Learnings
- Difference between Jenkins controller and agent
- Importance of Linux-based CI/CD pipelines
- Docker image build and push automation
- Kubernetes deployment using Helm
- Troubleshooting real Jenkins, Git, and Kubernetes issues

---

## Status
✅ CI/CD pipeline is fully working and green  
✅ Docker image is built, pushed, and deployed successfully  

---

## Next Steps
- Separate CI and CD stages
- Add environment-specific deployments
- Move the Linux agent to AWS EC2
