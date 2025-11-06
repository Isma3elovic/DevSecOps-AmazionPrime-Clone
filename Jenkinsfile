pipeline {
    agent any

    tools {
        jdk 'JDK'
        nodejs 'NodeJS'
    }

    environment {
        SCANNER_HOME = tool 'SonarQube Scanner'
        IMAGE_NAME = "isma3elovic/prime"
        IMAGE_TAG  = "latest"
        KUBE_CONFIG = "/var/jenkins_home/.kube/config"
    }

    stages {

        stage('1. Git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Isma3elovic/DevSecOps-AmazionPrime-Clone.git'
            }
        }

        stage('2. Sonar Analysis') {
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

        stage('4. Node npm install') {
            steps {
                sh "npm install"
            }
        }

        stage('5. Trivy Scan') {
            steps {
                sh "trivy fs . > trivy-scan.txt || true"
            }
        }

        stage('6. Docker Build and Push') {
            steps {
                script {
                    def app = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")

                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        app.push()
                    }
                }
            }
        }

        stage("7. Configure Prometheus & Grafana") {
            steps {
                script {
                    sh """
                    export KUBECONFIG=${KUBE_CONFIG}

                    helm repo add stable https://charts.helm.sh/stable || true
                    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                    helm repo update

                    # Create namespace if missing
                    if ! kubectl get namespace monitoring > /dev/null 2>&1; then
                        kubectl create namespace monitoring
                    fi

                    # Install Prometheus + Grafana via kube-prometheus-stack
                    if ! helm status kube-prom-stack -n monitoring > /dev/null 2>&1; then
                        helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
                            -n monitoring \
                            --set grafana.enabled=true \
                            --set grafana.service.type=NodePort \
                            --set grafana.adminPassword='admin123'
                    fi
                    """
                }
            }
        }

        stage("8. Configure ArgoCD") {
            steps {
                script {
                    sh """
                    export KUBECONFIG=${KUBE_CONFIG}

                    # Create namespace if missing
                    if ! kubectl get namespace argocd > /dev/null 2>&1; then
                        kubectl create namespace argocd
                    fi

                    # Install ArgoCD only if server deployment does not exist
                    if ! kubectl get deploy argocd-server -n argocd > /dev/null 2>&1; then
                        kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
                    fi
                    """
                }
            }
        }

        stage("9. Install Nginx Ingress Controller") {
            steps {
                script {
                    sh """
                    export KUBECONFIG=${KUBE_CONFIG}

                    # Create ingress namespace
                    if ! kubectl get namespace ingress-nginx > /dev/null 2>&1; then
                        kubectl create namespace ingress-nginx
                    fi

                    # Install Nginx ingress if missing
                    if ! helm status ingress-nginx -n ingress-nginx > /dev/null 2>&1; then
                        helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx || true
                        helm repo update
                        helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx
                    fi
                    """
                }
            }
        }

        stage('10. Deploy Application & Ingress') {
            steps {
                script{
                sh '''
    export KUBECONFIG=$KUBE_CONFIG

    NAMESPACE=$(grep 'namespace:' $WORKSPACE/k8s_files/deployment.yaml | awk '{print $2}')
    if [ -z "$NAMESPACE" ]; then
        NAMESPACE=amazon-prime
    fi

    if ! kubectl get namespace $NAMESPACE > /dev/null 2>&1; then
        kubectl create namespace $NAMESPACE
    fi

    kubectl apply -f $WORKSPACE/k8s_files/deployment.yaml -n $NAMESPACE
    kubectl apply -f $WORKSPACE/k8s_files/service.yaml -n $NAMESPACE

    kubectl rollout restart deployment amazon-prime -n $NAMESPACE

    if [ -f $WORKSPACE/k8s_files/ingress.yaml ]; then
        kubectl apply -f $WORKSPACE/k8s_files/ingress.yaml
    fi
'''

                }
            }
        }

    }
}
