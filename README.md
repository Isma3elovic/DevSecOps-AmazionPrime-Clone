# 🎬 Amazon Prime Clone Deployment Project

## 📘 Project Overview

This project demonstrates the deployment of an **Amazon Prime Clone** using a modern **DevOps CI/CD pipeline**.
It showcases the integration of automation, security, and monitoring tools for end-to-end deployment.

### 🧰 Tools Used

* **GitHub** → Source code management
* **Jenkins** → Continuous Integration & Continuous Deployment (CI/CD)
* **SonarQube** → Static code analysis and quality gate enforcement
* **NPM** → NodeJS dependency and build management
* **Aqua Trivy** → Security vulnerability scanning
* **Docker** → Application containerization
* **DockerHub** → Image registry for Docker images
* **Minikube** → Local Kubernetes cluster for app deployment
* **ArgoCD** → GitOps-based Continuous Deployment
* **Prometheus & Grafana** → Monitoring and alerting stack

---

## ⚙️ Prerequisites

Before you begin, install the following on your system:

### 🐳 Docker

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
docker --version
```

### 🧩 Docker Compose

```bash
sudo apt install docker-compose -y
docker-compose --version
```

### 🧠 Git

```bash
sudo apt install git -y
git --version
```

### 💻 Visual Studio Code (optional)

```bash
sudo snap install code --classic
```

---

## 🏗️ Infrastructure Setup Using Docker Compose

### 1️⃣ Clone the Repository

```bash
git clone https://github.com/Isma3elovic/DevSecOps-AmazionPrime-Clone.git
cd DevSecOps-AmazionPrime-Clone
code .   # optional: open project in VS Code
```

### 2️⃣ Run Docker Containers

```bash
docker-compose up -d
```

This command launches containers for **Jenkins** and **SonarQube**.

---

## 🔍 SonarQube Configuration

1. **Login Credentials:**
   Username → `admin`
   Password → `admin`

2. **Generate SonarQube Token:**
   Navigate to:
   `Administration → Security → Users → Tokens`

   * Generate a new token.
   * Save it for Jenkins integration.

---

## 🤖 Jenkins Configuration

### 1️⃣ Add Jenkins Credentials

Add the following credentials under:
**Manage Jenkins → Credentials → System → Global credentials**

* SonarQube Token
* DockerHub Username & Password

### 2️⃣ Install Required Plugins

Go to **Manage Jenkins → Plugins** and install:

* SonarQube Scanner
* NodeJS
* Docker
* Prometheus Metrics

### 3️⃣ Configure Global Tools

In **Manage Jenkins → Global Tool Configuration**, set up:

* JDK 17
* NodeJS
* SonarQube Scanner
* Docker

---

## 🚀 CI/CD Pipeline Overview

### 🧾 Pipeline Stages

1. **Git Checkout** → Clone source code from GitHub
2. **SonarQube Analysis** → Run static code analysis
3. **Quality Gate** → Verify code quality thresholds
4. **Install NPM Dependencies** → Install NodeJS packages
5. **Trivy Security Scan** → Scan for vulnerabilities
6. **Docker Build** → Build Docker image
7. **Push to DockerHub** → Push image to registry
8. **Image Cleanup** → Remove old images

---

## 🧱 Jenkins Build Pipeline Script

```groovy
pipeline {
    agent any
    
    tools {
        jdk 'JDK'
        nodejs 'NodeJS'
    }
    
    environment {
        SCANNER_HOME = tool 'SonarQube Scanner'
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
    }
    
    stages {
        stage('1. Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Isma3elovic/DevSecOps-AmazionPrime-Clone.git'
            }
        }
        
        stage('2. SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=amazon-prime \
                    -Dsonar.projectKey=amazon-prime
                    """
                }
            }
        }
        
        stage('3. Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }
        
        stage('4. Install npm') {
            steps {
                sh "npm install"
            }
        }
        
        stage('5. Trivy Scan') {
            steps {
                sh "trivy fs . > trivy.txt"
            }
        }
        
        stage('6. Build Docker Image') {
            steps {
                sh "docker build -t isma3elovic/amazon-prime:latest ."
            }
        }
        
        stage('7. Push Image to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push isma3elovic/amazon-prime:latest
                    """
                }
            }
        }
        
        stage('8. Cleanup Images') {
            steps {
                sh "docker rmi isma3elovic/amazon-prime:latest || true"
            }
        }
    }
}
```

---

## 🚢 Continuous Deployment with ArgoCD

### 1️⃣ Install Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start
```

### 2️⃣ Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### 3️⃣ Install Prometheus & Grafana

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create namespace monitoring
helm install monitor prometheus-community/kube-prometheus-stack -n monitoring
```

---

## 🧹 Cleanup Resources

To remove all resources and free up space:

```bash
kubectl delete namespace argocd --ignore-not-found
kubectl delete namespace monitoring --ignore-not-found
minikube delete
docker system prune -af
```

---

## 📄 Additional Information

For a complete explanation of each component, refer to the project documentation included in this repository.

---
