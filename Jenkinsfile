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
    kubectl apply -f $WORKSPACE/k8s_files/prime-svc.yaml -n $NAMESPACE

    kubectl rollout restart deployment amazon-prime -n $NAMESPACE

    
'''

                }
            }
        }

    }
}
