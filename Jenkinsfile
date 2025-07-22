pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "sushilicp/my-web-app"
        DOCKER_TAG = "${env.BUILD_ID ?: 'latest'}"
        CONTAINER_NAME = "my-web-app-${env.BUILD_NUMBER}"
        GOOGLE_CHAT_WEBHOOK = credentials('google-chat-webhook')
        DEPLOYMENT_URL = "http://localhost:8088"
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        HOST_PORT = "8088"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/anishSub/genkins.git',
                    credentialsId: '' // Add GitHub credentials ID here if private repo
            }
        }

        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {
                        sh """
                            echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh """
                    docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                    docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                    docker push ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh """
                        # Stop container using HOST_PORT if running
                        CONTAINER_USING_PORT=\$(docker ps --filter "publish=${HOST_PORT}" --format "{{.Names}}" | head -n 1)

                        if [ -n "\$CONTAINER_USING_PORT" ]; then
                            echo "Stopping container using port ${HOST_PORT}: \$CONTAINER_USING_PORT"
                            docker stop \$CONTAINER_USING_PORT || true
                            docker rm \$CONTAINER_USING_PORT || true
                        fi

                        # Remove container if same name already exists
                        if docker ps -a --format '{{.Names}}' | grep -Eq "^${CONTAINER_NAME}\$"; then
                            docker stop ${CONTAINER_NAME} || true
                            docker rm ${CONTAINER_NAME} || true
                        fi

                        # Run new container
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${HOST_PORT}:80 \
                            ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }

        success {
            script {
                def message = """
                ✅ *Deployment Successful*
                *Build:* #${env.BUILD_NUMBER}
                *Image:* ${DOCKER_IMAGE}:${DOCKER_TAG}
                *URL:* ${DEPLOYMENT_URL}
                """
                sendGoogleChatNotification(message)
            }
        }

        failure {
            script {
                def logs = sh(script: "docker logs --tail 50 ${CONTAINER_NAME} || echo 'No logs available'", returnStdout: true).trim()
                def message = """
                ❌ *Deployment Failed*
                *Build:* #${env.BUILD_NUMBER}
                *Error:* ${currentBuild.currentResult}
                *Logs:* ${logs}
                """
                sendGoogleChatNotification(message)
            }
        }
    }
}

def sendGoogleChatNotification(String message) {
    def escapedMessage = message.replace('"', '\\"')
    def payload = """{"text": "${escapedMessage}"}"""

    sh """
        curl -X POST \
        -H "Content-Type: application/json" \
        -d '${payload}' \
        "${GOOGLE_CHAT_WEBHOOK}" || echo "Notification failed"
    """
}
