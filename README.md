Markdown

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

🧩 Docker Compose

Bash

sudo apt install docker-compose -y
docker-compose --version

🧠 Git

Bash

sudo apt install git -y
git --version

💻 Visual Studio Code (optional)

Bash

sudo snap install code --classic

🏗️ Infrastructure Setup Using Docker Compose

1️⃣ Clone the Repository

Bash

git clone [https://github.com/Isma3elovic/DevSecOps-AmazionPrime-Clone.git](https://github.com/Isma3elovic/DevSecOps-AmazionPrime-Clone.git)
cd DevSecOps-AmazionPrime-Clone
code .   # optional: open project in VS Code

2️⃣ Run Docker Containers

Bash

docker-compose up -d

This command launches containers for Jenkins and SonarQube.

3️⃣ Configure Jenkins Container for CI/CD Tools

Access the Jenkins container as the root user to install the necessary command-line tools and configure Kubernetes access. Replace <JENKINS_CONTAINER_ID> with your running container's ID.

A. Access the Jenkins Container

Bash

docker exec -u 0 -it <JENKINS_CONTAINER_ID> bash

B. Install Trivy (Security Scanner)

Run these commands inside the container:
Bash

apt update
apt install -y wget apt-transport-https gnupg lsb-release
wget -qO - [https://aquasecurity.github.io/trivy-repo/deb/public.key](https://aquasecurity.github.io/trivy-repo/deb/public.key) | gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] [https://aquasecurity.github.io/trivy-repo/deb](https://aquasecurity.github.io/trivy-repo/deb) $(lsb_release -sc) main" > /etc/apt/sources.list.d/trivy.list
apt update 
apt install -y trivy

C. Install Docker CLI

Run these commands inside the container:
Bash

apt-get update -y
apt-get install -y ca-certificates curl gnupg
curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update -y
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

D. Install Helm

Run these commands inside the container:
Bash

apt-get update && apt-get install -y curl apt-transport-https 
curl [https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3) | bash

E. Configure Docker Access and Permissions

Run these commands inside the container:
Bash

usermod -aG docker ubuntu
chmod 777 /var/run/docker.sock 

F. Setup Minikube/Kubernetes Configuration

Run these commands on your Host Machine (outside the container):

    Connect Networks (Host):
    Bash

docker network connect minikube <JENKINS_CONTAINER_ID>

Copy Kubeconfig (Host):
Bash

    docker exec -it <JENKINS_CONTAINER_ID> mkdir -p /var/jenkins_home/.kube
    docker cp ~/.kube/config <JENKINS_CONTAINER_ID>:/var/jenkins_home/.kube/config

🔍 SonarQube Configuration

Login Credentials:

    Username → admin

    Password → admin

Generate SonarQube Token:

    Navigate to: Administration → Security → Users → Tokens

    Generate a new token.

    Save it for Jenkins integration.

🤖 Jenkins Configuration

1️⃣ Add Jenkins Credentials

Add the following credentials under: Manage Jenkins → Credentials → System → Global credentials

    Secret Text: SonarQube Token (ID: sonar-token)

    Username with password: DockerHub Username & Password (ID: dockerhub-creds)

2️⃣ Install Required Plugins

Go to Manage Jenkins → Plugins and install:

    SonarQube Scanner

    NodeJS

    Docker

    Prometheus Metrics

3️⃣ Configure Global Tools

In Manage Jenkins → Global Tool Configuration, set up:

    JDK 17 (Name: JDK)

    NodeJS (Name: NodeJS)

    SonarQube Scanner (Name: SonarQube Scanner)

    Docker

🚀 CI/CD Pipeline Overview

🧾 Pipeline Stages

    Git Checkout → Clone source code from GitHub

    SonarQube Analysis → Run static code analysis

    Quality Gate → Verify code quality thresholds

    Install NPM Dependencies → Install NodeJS packages

    Trivy Security Scan → Scan for vulnerabilities

    Docker Build → Build Docker image

    Push to DockerHub → Push image to registry

    Image Cleanup → Remove old images

🧱 Jenkins Build Pipeline Script

Groovy

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
                git branch: 'main', url: '[https://github.com/Isma3elovic/DevSecOps-AmazionPrime-Clone.git](https://github.com/Isma3elovic/DevSecOps-AmazionPrime-Clone.git)'
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

🚢 Continuous Deployment with ArgoCD

1️⃣ Install Minikube

Bash

curl -LO [https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64](https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64)
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start

2️⃣ Install ArgoCD

Bash

kubectl create namespace argocd
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

3️⃣ Install Prometheus & Grafana

Bash

helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
kubectl create namespace monitoring
helm install monitor prometheus-community/kube-prometheus-stack -n monitoring

🧹 Cleanup Resources

To remove all resources and free up space:
Bash

kubectl delete namespace argocd --ignore-not-found
kubectl delete namespace monitoring --ignore-not-found
minikube delete
docker system prune -af
