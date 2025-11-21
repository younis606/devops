pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "younis606/result:${GIT_COMMIT}"
        HELM_RELEASE = "result-app"
        HELM_CHART_PATH = "./Helm Chart/result"  // مسار Helm Chart الخاص بالخدمة result
        KUBECONFIG_DEV = credentials('kubeconfig-dev')
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Check Workspace') {
            steps {
                sh "ls -R"
            }
        }

        stage('Build Docker Image (result)') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE} -f result/Dockerfile result/"
            }
        }

        stage('Trivy Scan (result)') {
            steps {
                script {
                    trivyScan(
                        imageName: "${DOCKER_IMAGE}",
                        severity: "HIGH,CRITICAL",
                        exitCode: 0
                    )
                }
            }
        }

        stage('Push Docker Image (result)') {
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
                  --set image.repository=younis606/result \
                  --set image.tag=${GIT_COMMIT}
                """
            }
        }

        stage('Smoke Test (result)') {
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
            echo "Result service pipeline completed successfully!"
        }
    }
}
