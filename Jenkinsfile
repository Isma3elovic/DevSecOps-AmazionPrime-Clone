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

    triggers {
        // Automatically trigger when GitHub webhook fires (on push)
        githubPush()
    }

    stages {

        stage('1. Git checkout') {
            when {
                branch 'main'
            }
            steps {
                git branch: 'main', url: 'https://github.com/Isma3elovic/DevSecOps-AmazionPrime-Clone.git'
            }
        }

        stage('2. Sonar Analysis') {
            when {
                branch 'main'
            }
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
            when {
                branch 'main'
            }
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }

        stage('4. Node npm install') {
            when {
                branch 'main'
            }
            steps {
                sh "npm install"
            }
        }

        stage('5. Trivy Scan') {
            when {
                branch 'main'
            }
            steps {
                sh "trivy fs . > trivy-scan.txt || true"
            }
        }

        stage('6. Docker Build and Push') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def app = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")

                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        app.push()
                    }
                }
            }
        }

        stage('7. Deploy Application & Ingress') {
            when {
                branch 'main'
            }
            steps {
                script {
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
                        docker system prune -f
                    '''
                }
            }
        }
    }
}
