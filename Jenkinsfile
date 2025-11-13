
 pipeline {
    agent any
    
    environment {
        // Docker configuration
        DOCKER_IMAGE = 'sanjaybsingh/docker-flask-app'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        DOCKER_REGISTRY = 'docker.io' // Change to your registry (e.g., 'your-registry.azurecr.io')
        DOCKER_REGISTRY_CREDENTIAL = 'sanjaybsingh' // Jenkins credential ID
        
        // Kubernetes configuration (optional)
        //KUBECONFIG_CREDENTIAL = 'kubeconfig' // Jenkins credential ID for kubeconfig file
        //K8S_NAMESPACE = 'default'
        
        // Application configuration
        APP_NAME = 'flask-docker-app'
        APP_PORT = '5000'
        DEPLOY_ENV = "${env.BRANCH_NAME == 'main' ? 'production' : 'staging'}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "Checking out code from ${env.GIT_BRANCH}"
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    dockerImage = docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    
                    // Also tag as latest for the current branch
                    sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    echo "Running tests inside Docker container"
                    // Run a test container
                    sh """
                        docker run --rm ${DOCKER_IMAGE}:${DOCKER_TAG} python -c "
from app import app
import sys
print('Testing Flask app import...')
try:
    with app.test_client() as client:
        response = client.get('/')
        assert response.status_code == 200
        assert b'Hello, Docker!' in response.data
        print('✓ All tests passed!')
        sys.exit(0)
except Exception as e:
    print(f'✗ Test failed: {e}')
    sys.exit(1)
"
                    """
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    echo "Scanning Docker image for vulnerabilities"
                    // Optional: Use Trivy or similar tool
                    sh """
                        # Install trivy if not available (uncomment if needed)
                        # curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
                        
                        # Run security scan (uncomment if trivy is installed)
                        # trivy image --severity HIGH,CRITICAL ${DOCKER_IMAGE}:${DOCKER_TAG}
                        
                        echo "Security scan stage - configure with your preferred tool"
                    """
                }
            }
        }
        
        stage('Push to Registry') {
            when {
                // Only push for main and feature branches (not for PRs)
                expression { env.CHANGE_ID == null }
            }
            steps {
                script {
                    echo "Pushing image to Docker registry"
                    docker.withRegistry("https://${DOCKER_REGISTRY}", "${DOCKER_REGISTRY_CREDENTIAL}") {
                        // Push with build number tag
                        sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}"
                        sh "docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}"
                        
                        // Push latest tag for main branch
                        if (env.BRANCH_NAME == 'main') {
                            sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:latest"
                            sh "docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:latest"
                        }
                    }
                }
            }
        }
        
        
    post {
        success {
            echo "✓ Pipeline completed successfully!"
            echo "Image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
            echo "Deployed to: ${DEPLOY_ENV}"
            
            // Optional: Send notification
            // slackSend(color: 'good', message: "Deployment succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
        
        failure {
            echo "✗ Pipeline failed!"
            // Optional: Send notification
            // slackSend(color: 'danger', message: "Deployment failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
        
        always {
            echo "Cleaning up..."
            // Clean up Docker images to save space
            sh """
                docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true
                docker system prune -f || true
            """
        }
    }
}

// Deployment functions

def deployToDocker() {
    echo "Deploying to local Docker host"
    sh """
        # Stop and remove old container if exists
        docker stop ${APP_NAME}-${DEPLOY_ENV} || true
        docker rm ${APP_NAME}-${DEPLOY_ENV} || true
        
        # Run new container
        docker run -d \
            --name ${APP_NAME}-${DEPLOY_ENV} \
            -p ${APP_PORT}:5000 \
            --restart unless-stopped \
            ${DOCKER_IMAGE}:${DOCKER_TAG}
        
        echo "Container ${APP_NAME}-${DEPLOY_ENV} is running on port ${APP_PORT}"
    """
}

def deployToKubernetes() {
    echo "Deploying to Kubernetes cluster"
    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL}", variable: 'KUBECONFIG')]) {
        sh """
            # Update deployment with new image
            kubectl set image deployment/${APP_NAME} \
                ${APP_NAME}=${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG} \
                -n ${K8S_NAMESPACE} \
                --kubeconfig=\$KUBECONFIG
            
            # Wait for rollout to complete
            kubectl rollout status deployment/${APP_NAME} \
                -n ${K8S_NAMESPACE} \
                --kubeconfig=\$KUBECONFIG \
                --timeout=5m
            
            echo "Kubernetes deployment completed"
        """
    }
}