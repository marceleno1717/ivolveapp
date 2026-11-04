pipeline {
    agent any

    environment {
        IMAGE_NAME = 'marceleno17/ivolveapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials'
        KUBECONFIG_CREDENTIALS_ID = 'kubeconfig-credentials'
    }

    stages {
        stage('1. Build App') {
            steps {
                echo "Installing application dependencies..."
                sh 'npm install'
            }
        }

        stage('2. Build Docker Image') {
            steps {
                echo "Building Docker image ${IMAGE_NAME}:${IMAGE_TAG}..."
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('3. Scan Docker Image') {
            steps {
                echo "Scanning Docker image with Trivy..."
                sh "trivy image --severity HIGH,CRITICAL --exit-code 0 ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('4. Push Image to DockerHub') {
            steps {
                echo "Pushing Docker image to DockerHub..."
                withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('5. Delete Image Locally') {
            steps {
                echo "Cleaning up local Docker images..."
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('6. Update Image in Deployment File') {
            steps {
                echo "Updating deployment.yaml with image tag ${IMAGE_TAG}..."
                sh "sed -i 's|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|' deployment.yaml"
            }
        }

        stage('7. Deploy to Kubernetes Cluster') {
            steps {
                echo "Applying manifests to Kubernetes cluster..."
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
                    sh 'kubectl apply -f deployment.yaml'
                    sh 'kubectl rollout status deployment/ivolveapp -n ivolve --timeout=120s'
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline run completed."
            sh 'docker logout || true'
        }
        success {
            echo "Pipeline succeeded! Application deployed with image ${IMAGE_NAME}:${IMAGE_TAG}."
        }
        failure {
            echo "Pipeline failed! Please inspect logs for details."
        }
    }
}
