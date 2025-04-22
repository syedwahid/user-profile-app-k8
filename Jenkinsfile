pipeline {
    agent any

    triggers {
        pollSCM('* * * * *')
    }

    environment {
        DOCKER_IMAGE = "syedwahid/user-profile-app-k8"
        GIT_REPO = "https://github.com/syedwahid/user-profile-app-k8"
        CONTAINER_NAME = "user-profile-app-k8"
        TEST_PORT = "3001"
        // Add Kubernetes namespace (adjust as needed)
        K8S_NAMESPACE = "default"
    }

    stages {
        // Stage 1: Checkout
        stage('Checkout') {
            steps {
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

        // Stage 2: Run Tests
        stage('Test Phone Number') {
            steps {
                sh '''
                export PORT=${TEST_PORT}
                npm install
                npm test
                '''
            }
        }

        // Stage 3: Cleanup Docker
        stage('Clean Docker Environment') {
            steps {
                sh '''
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
                docker rmi -f $(docker images -q ${DOCKER_IMAGE}) || true
                docker image prune -f
                '''
            }
        }

        // Stage 4: Build Image
        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                '''
            }
        }

        // Stage 5: Push to Docker Hub
        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}
                    docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        // Stage 6: Deploy to Kubernetes (REPLACED "Run Docker Container")
        stage('Deploy to Kubernetes') {
            steps {
                // Ensure kubectl is configured for your local cluster (e.g., Minikube)
                sh '''
                # Substitute variables in Kubernetes manifests
                envsubst < kubernetes/deployment.yaml.template > kubernetes/deployment.yaml
                # Apply Kubernetes resources
                kubectl apply -f kubernetes/ -n ${K8S_NAMESPACE}
                '''
            }
        }
    }

    post {
        always {
            sh 'docker system df'
            // Optional: Add Kubernetes status checks
            sh 'kubectl get pods -n ${K8S_NAMESPACE}'
        }
        success {
            slackSend message: "✅ Pipeline SUCCESS - ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
        failure {
            slackSend message: "❌ Pipeline FAILED - ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
    }
}
