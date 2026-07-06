pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USER = 'sushmitacb'
        IMAGE_NAME = 'sushmitacb/crochetcraft-app'
        REGISTRY_CREDENTIALS_ID = 'docker-hub-credentials'
    }

    stages {
        stage('Build JAR') {
            steps {
                echo "🔨 Building Spring Boot application..."
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    echo "📦 Building Docker image ${IMAGE_NAME}:${BUILD_NUMBER} and latest..."
                    sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
                    sh "docker build -t ${IMAGE_NAME}:latest ."

                    echo "🔐 Logging into Docker Hub and pushing image..."
                    withCredentials([usernamePassword(credentialsId: REGISTRY_CREDENTIALS_ID, usernameVariable: 'USER', passwordVariable: 'PASSWORD')]) {
                        sh "(echo \${PASSWORD} | docker login -u \${USER} --password-stdin ${DOCKER_REGISTRY}) || echo 'Using system docker config...'"
                        sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
                        sh "docker push ${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "☸️ Deploying MySQL database and Spring Boot app to Kubernetes cluster..."
                    // Apply MySQL deployment and PV/PVC
                    sh "kubectl apply -f k8s/mysql-deployment.yaml"
                    
                    // Apply Spring Boot application deployment and service
                    sh "kubectl apply -f k8s/app-deployment.yaml"
                    
                    // Update deployment with the newly built Docker image tag
                    sh "kubectl set image deployment/springboot-app springboot-app=${IMAGE_NAME}:${BUILD_NUMBER}"
                }
            }
        }

        stage('Verify Kubernetes Deployment') {
            steps {
                echo "🧪 Verifying rollouts and container status..."
                sh "kubectl rollout status deployment/mysql-container --timeout=300s"
                sh "kubectl rollout status deployment/springboot-app --timeout=300s"
                sh "kubectl get pods,svc -l app=crochetcraft -o wide"
            }
        }
    }

    post {
        success {
            echo "✅ Spring Boot app and MySQL deployed successfully to Kubernetes!"
        }
        failure {
            echo "❌ Pipeline execution failed. Check console logs for details."
        }
        always {
            echo "ℹ️ Pipeline completed."
        }
    }
}
