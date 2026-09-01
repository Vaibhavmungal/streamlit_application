pipeline {
    agent any

    parameters {
        string(
            name: 'DOCKER_IMAGE_NAME',
            defaultValue: 'vaibhavmungal/streamlit-app',
            description: 'Docker Hub repository name (e.g., username/repository)'
        )
        booleanParam(
            name: 'DEPLOY_TO_K8S',
            defaultValue: true,
            description: 'Whether to deploy to Kubernetes cluster after pushing image'
        )
    }

    environment {
        DOCKER_IMAGE = "${params.DOCKER_IMAGE_NAME}"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
        KUBECONFIG_CREDENTIALS_ID = 'k8s-kubeconfig'
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out source code from Git..."
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                echo "Building Docker image: ${DOCKER_IMAGE}:${IMAGE_TAG} and latest..."
                script {
                    sh """
                        docker build \
                            -t ${DOCKER_IMAGE}:${IMAGE_TAG} \
                            -t ${DOCKER_IMAGE}:latest .
                    """
                }
            }
        }

        stage('Docker Hub Push') {
            steps {
                echo "Authenticating and pushing images to Docker Hub..."
                script {
                    withCredentials([usernamePassword(
                        credentialsId: "${env.DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                            docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                            docker push ${DOCKER_IMAGE}:latest
                            docker logout
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression { return params.DEPLOY_TO_K8S }
            }
            steps {
                echo "Deploying application to Kubernetes cluster..."
                script {
                    // Update deployment manifest with the newly built image tag
                    sh """
                        sed -i.bak "s|image: .*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|g" k8s/deployment.yaml
                        cat k8s/deployment.yaml | grep "image:"
                    """

                    // Deploy using kubeconfig credentials if configured, otherwise using agent's ambient kubectl
                    try {
                        withCredentials([file(credentialsId: "${env.KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
                            sh "kubectl apply -f k8s/"
                        }
                    } catch (Exception e) {
                        echo "Kubeconfig credential not found or failed, attempting local agent kubectl..."
                        sh "kubectl apply -f k8s/"
                    }
                }
            }
        }

        stage('Verify Rollout') {
            when {
                expression { return params.DEPLOY_TO_K8S }
            }
            steps {
                echo "Verifying rollout status..."
                script {
                    try {
                        withCredentials([file(credentialsId: "${env.KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
                            sh "kubectl rollout status deployment/streamlit-app --timeout=120s"
                            sh "kubectl get pods,svc -l app=streamlit-app"
                        }
                    } catch (Exception e) {
                        sh "kubectl rollout status deployment/streamlit-app --timeout=120s"
                        sh "kubectl get pods,svc -l app=streamlit-app"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up local Docker images..."
            sh """
                docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG} ${DOCKER_IMAGE}:latest || true
            """
        }
        success {
            echo "Pipeline completed successfully! Streamlit app is live on K8s."
        }
        failure {
            echo "Pipeline failed! Please check stage logs for details."
        }
    }
}
