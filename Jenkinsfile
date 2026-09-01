pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'rutwik02/streamlit-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        DOCKER_CREDENTIALS_ID = 'DOCKER_HUB_CREDS'
    }

    stages {
        stage('Cleanup Disk Space') {
            steps {
                echo "Cleaning up dangling Docker cache to free disk space..."
                sh '''
                    docker image prune -f || true
                    docker builder prune -f || true
                '''
            }
        }

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

        stage('Run Container') {
            steps {
                echo "Deploying and running Streamlit container on port 8501..."
                script {
                    sh """
                        docker stop streamlit-app || true
                        docker rm streamlit-app || true
                        docker run -d -p 8501:8501 --restart unless-stopped --name streamlit-app ${DOCKER_IMAGE}:latest
                        sleep 3
                        docker ps | grep streamlit-app
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning dangling images and builder cache..."
            sh '''
                docker image prune -f || true
                docker builder prune -f || true
            '''
        }
        success {
            echo "Pipeline completed successfully! Streamlit app is live at http://<server-ip>:8501"
        }
        failure {
            echo "Pipeline failed! Please check stage logs for details."
        }
    }
}
