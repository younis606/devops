pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "younis606/vote-app:${GIT_COMMIT}"
        HELM_RELEASE = "voting-app"
        HELM_CHART_PATH = "./Helm_Chart/vote-app"
        KUBECONFIG_DEV = credentials('kubeconfig-dev')
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.DOCKER_IMAGE = "younis606/vote-app:${env.GIT_COMMIT}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Trivy Scan') {
            steps {
                sh "trivy image --severity HIGH,CRITICAL ${DOCKER_IMAGE} || true"
            }
        }

        stage('Push Docker Image') {
            steps {
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

        stage('Deploy via Helm (Dev)') {
            steps {
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
                sh """
                HTTP_STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://vote.local/)
                if [ "\$HTTP_STATUS" -ne 200 ]; then
                    exit 1
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
