# 🚀 Trend DevOps CI/CD Pipeline Project

A complete end-to-end DevOps pipeline that automates building, pushing, and deploying a web application to AWS EKS using Jenkins, Docker, Terraform, and Kubernetes — with Prometheus and Grafana for monitoring.

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [1. Application Setup](#1-application-setup)
  - [2. Dockerize the Application](#2-dockerize-the-application)
  - [3. Terraform Infrastructure](#3-terraform-infrastructure)
  - [4. Kubernetes Setup on EKS](#4-kubernetes-setup-on-eks)
  - [5. Jenkins CI/CD Pipeline](#5-jenkins-cicd-pipeline)
  - [6. GitHub Webhook Integration](#6-github-webhook-integration)
  - [7. Monitoring with Prometheus & Grafana](#7-monitoring-with-prometheus--grafana)
- [Pipeline Explanation](#pipeline-explanation)
- [Application URLs](#application-urls)
- [Screenshots](#screenshots)

---

## Architecture Overview

```
Developer pushes code
        ↓
   GitHub Repository
        ↓
  Jenkins (Webhook Trigger)
        ↓
  Docker Build & Push → DockerHub
        ↓
  kubectl apply → AWS EKS
        ↓
  App live on LoadBalancer
        ↓
  Prometheus + Grafana (Monitoring)
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| **GitHub** | Version control and webhook trigger |
| **Jenkins** | CI/CD automation server |
| **Docker** | Containerization |
| **DockerHub** | Container image registry |
| **Terraform** | Infrastructure as Code (IaC) |
| **AWS EKS** | Managed Kubernetes cluster |
| **Kubernetes** | Container orchestration |
| **Prometheus** | Metrics collection |
| **Grafana** | Monitoring dashboards |
| **AWS EC2** | Jenkins server hosting |

---

## Project Structure

```
trend-devops-project/
├── dist/                    # Built application files
├── terraform/
│   └── main.tf              # Terraform infra definition (VPC, IAM, EKS)
├── Dockerfile               # Docker image definition
├── Jenkinsfile              # Declarative pipeline script
├── deployment.yaml          # Kubernetes deployment manifest
├── service.yaml             # Kubernetes LoadBalancer service
└── README.md
```

---

## Prerequisites

- AWS Account with IAM user (`project_1`) having EKS, EC2, IAM permissions
- EC2 instance (Ubuntu) with public IP
- DockerHub account
- GitHub account

---

## Setup Instructions

### 1. Application Setup

Clone the source repository and set up on EC2:

```bash
# SSH into EC2
ssh -i your-key.pem ubuntu@13.203.93.113

# Clone the repo
git clone https://github.com/GOPINATH0926/trend-devops-project.git
cd trend-devops-project
```

---

### 2. Dockerize the Application

**Dockerfile:**

```dockerfile
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY dist/ /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Build and push:**

```bash
docker build -t trend-app .
docker tag trend-app gopinathsiva2605/trend-app:latest
docker login
docker push gopinathsiva2605/trend-app:latest
```

---

### 3. Terraform Infrastructure

Terraform provisions the AWS infrastructure using the existing default VPC and IAM roles for EKS.

**Resources created:**
- IAM Role for EKS Cluster (`trend-eks-role`)
- IAM Role for Node Group (`trend-node-role`)
- EKS Cluster (`trend-cluster`) in `ap-south-1`
- Managed Node Group with 2x `t3.medium` instances

```bash
cd terraform/
aws configure   # Set Access Key, Secret, region: ap-south-1
terraform init
terraform plan
terraform apply -auto-approve
```

**Connect kubectl to EKS:**

```bash
aws eks update-kubeconfig --region ap-south-1 --name trend-cluster
kubectl get nodes
```

---

### 4. Kubernetes Setup on EKS

**deployment.yaml** — deploys 2 replicas of the app:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trend-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: trend
  template:
    metadata:
      labels:
        app: trend
    spec:
      containers:
      - name: trend
        image: gopinathsiva2605/trend-app:latest
        ports:
        - containerPort: 80
```

**service.yaml** — exposes the app via AWS LoadBalancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: trend-service
spec:
  type: LoadBalancer
  selector:
    app: trend
  ports:
  - port: 80
    targetPort: 80
```

**Deploy manually:**

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get svc trend-service   # Copy EXTERNAL-IP
```

---

### 5. Jenkins CI/CD Pipeline

**Jenkins installed on EC2 at:** `http://13.203.93.113:8080`

**Plugins installed:**
- Git Plugin
- GitHub Integration Plugin
- Pipeline Plugin
- Docker Pipeline Plugin
- Kubernetes CLI Plugin
- Credentials Binding Plugin

**DockerHub credentials added:**
- Go to Manage Jenkins → Credentials → Global
- Kind: Username with password
- ID: `dockerhub-creds`

**Jenkinsfile (Declarative Pipeline):**

```groovy
pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "gopinathsiva2605/trend-app:latest"
    }
    stages {
        stage('Clone') {
            steps {
                git branch: 'main', url: 'https://github.com/GOPINATH0926/trend-devops-project.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t trend-app .'
                sh "docker tag trend-app ${DOCKER_IMAGE}"
            }
        }
        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml'
                sh 'kubectl rollout status deployment/trend-deployment'
            }
        }
    }
    post {
        success { echo 'Deployment successful!' }
        failure { echo 'Build failed. Check logs.' }
    }
}
```

**Pipeline stages:**

| Stage | Description |
|---|---|
| Clone | Pulls latest code from GitHub |
| Build Docker Image | Builds and tags the Docker image |
| Push to DockerHub | Pushes image to `gopinathsiva2605/trend-app` |
| Deploy to Kubernetes | Applies manifests to EKS cluster |

---

### 6. GitHub Webhook Integration

Enables automatic pipeline trigger on every `git push`.

1. Go to GitHub repo → Settings → Webhooks → Add webhook
2. Payload URL: `http://13.203.93.113:8080/github-webhook/`
3. Content type: `application/json`
4. Event: Push event
5. In Jenkins → `trend-pipeline` → Configure → Build Triggers → enable **GitHub hook trigger for GITScm polling**

**Flow:**
```
git push → GitHub webhook → Jenkins auto-triggered → Build → Push → Deploy
```

---

### 7. Monitoring with Prometheus & Grafana

Installed via Helm on the EKS cluster:

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add Prometheus community chart
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus + Grafana stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# Expose Grafana externally
kubectl patch svc monitoring-grafana -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'

# Get Grafana URL
kubectl get svc monitoring-grafana -n monitoring
```

**Grafana Login:**
```bash
# Get credentials
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

**Key dashboards available:**
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Nodes
- Kubernetes / Pods
- Kubernetes / Deployments

---

## Pipeline Explanation

The CI/CD pipeline follows this automated flow:

1. Developer pushes code to the `main` branch on GitHub
2. GitHub sends a webhook POST to Jenkins at `:8080/github-webhook/`
3. Jenkins triggers the `trend-pipeline` automatically
4. **Clone stage** — Jenkins pulls the latest code
5. **Build stage** — Docker builds the image using the Dockerfile
6. **Push stage** — Image is tagged and pushed to DockerHub
7. **Deploy stage** — `kubectl apply` updates the running pods on EKS
8. Kubernetes performs a rolling update with zero downtime
9. Prometheus scrapes metrics; Grafana dashboards reflect the new deployment

---

## Application URLs

| Service | URL |
|---|---|
| Jenkins | http://13.203.93.113:8080 |
| Application (EC2) | http://13.203.93.113:3000 |
| DockerHub Image | https://hub.docker.com/r/gopinathsiva2605/trend-app |
| Grafana Dashboard | http://aef662bceb21743dfb169b07679c5a45-1412764752.ap-south-1.elb.amazonaws.com |

> **Application LoadBalancer ARN:** Run `kubectl get svc trend-service` to get the current external IP.

---

---

## Author

**Gopinath**
HCLTech | Cisco Jasper  
GitHub: [@GOPINATH0926](https://github.com/GOPINATH0926)
