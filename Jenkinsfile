pipeline {
    agent any

    environment {
        KUBECONFIG_DEV  = credentials('kubeconfig-dev')
        DOCKER_REGISTRY = 'localhost:5000'
        HELM_RELEASE    = 'voting-app'
        HELM_CHART_PATH = './helm/voting-app'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.DOCKER_IMAGE = "${env.DOCKER_REGISTRY}/voting-app:${env.GIT_COMMIT}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${env.DOCKER_IMAGE} ."
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                trivy image ${env.DOCKER_IMAGE} \
                  --severity CRITICAL,HIGH \
                  --exit-code 1 \
                  --format json -o trivy-image-results.json
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry([credentialsId: 'docker-hub-credentials', url: "http://${env.DOCKER_REGISTRY}"]) {
                        sh "docker push ${env.DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy via Helm (Dev)') {
            steps {
                sh """
                helm upgrade --install ${HELM_RELEASE} ${HELM_CHART_PATH} \
                  --namespace vote --create-namespace \
                  --kubeconfig ${KUBECONFIG_DEV} \
                  --values ${HELM_CHART_PATH}/values.yaml
                """
            }
        }

        stage('Smoke Test') {
            steps {
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

        stage('IaC Workflow / Prod Deployment via GitOps') {
            when {
                expression { env.BRANCH_NAME == 'main' }
            }
            steps {
                echo "Production deployment handled via GitOps automation (ArgoCD / FluxCD)"
            }
        }
    }

    post {
        always {
            echo "Pipeline completed"
        }

        failure {
            echo "Pipeline failed! Check logs above."
        }

        success {
            echo "Vote-app pipeline completed successfully!"
        }
    }
}
