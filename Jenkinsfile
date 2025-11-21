pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "younis606/vote-app:${GIT_COMMIT}"
        HELM_RELEASE = "voting-app"
        HELM_CHART_PATH = "./Helm Chart/vote-app"
        KUBECONFIG_DEV = credentials('kubeconfig-dev')
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo 'Cloning repository from GitHub...'
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image for Vote App...'
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    echo "Scanning Docker image for vulnerabilities..."
                
                    trivyScan(
                        imageName: "${DOCKER_IMAGE}",
                        severity: "HIGH,CRITICAL",
                        exitCode: 0
                    )
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    echo 'Pushing Docker image to Docker Hub...'
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'docker-hub-credentials',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh """
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker push ${DOCKER_IMAGE}
                        """
                    }
                }
            }
        }

        stage('Deploy via Helm (Dev)') {
            steps {
                echo 'Deploying to Kubernetes (Dev)...'
                sh """
                helm upgrade --install ${HELM_RELEASE} ${HELM_CHART_PATH} \
                  --namespace vote --create-namespace \
                  --kubeconfig ${KUBECONFIG_DEV} \
                  --values ${HELM_CHART_PATH}/values.yaml \
                  --set image.repository=younis606/vote-app \
                  --set image.tag=${GIT_COMMIT}
                """
            }
        }

        stage('Smoke Test') {
            steps {
                echo 'Running Smoke Test...'
                sh """
                HTTP_STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://vote.local/)
                if [ "\$HTTP_STATUS" -ne 200 ]; then
                    echo "Smoke test failed! Status code: \$HTTP_STATUS"
                    exit 1
                else
                    echo "Smoke test passed! Status code: \$HTTP_STATUS"
                fi
                """
            }
        }
    }

    post {
        always {
            echo "Pipeline finished"
        }
        failure {
            echo "Pipeline failed! Check logs above."
        }
        success {
            echo "Vote App pipeline completed successfully!"
        }
    }
}
