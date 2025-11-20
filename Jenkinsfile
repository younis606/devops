pipeline {
    agent any

    tools {
        nodejs 'nodejs-22-6-0'
    }

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

        stage('Install Dependencies') {
            steps {
                sh 'npm install --no-audit'
            }
        }

        stage('NPM Dependency Audit') {
            steps {
                sh 'npm audit --audit-level=critical || true'
            }
        }

        stage('Unit Testing') {
            steps {
                sh 'npm test || true'
            }
        }

        stage('Code Coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    sh 'npm run coverage'
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
                  --ignore-unfixed \
                  --format json \
                  -o trivy-image-results.json || true
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
                withEnv(["KUBECONFIG=${KUBECONFIG_DEV}"]) {
                    sh """
                    helm upgrade --install ${HELM_RELEASE} ${HELM_CHART_PATH} \
                      --namespace vote --create-namespace \
                      --values ${HELM_CHART_PATH}/values.yaml
                    """
                }
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

        stage('Update and Commit Image Tag (GitOps)') {
            when {
                expression { env.BRANCH_NAME?.startsWith('PR') }
            }
            steps {
                sh """
                rm -rf vote-app-gitops
                git clone -b main http://git-server:5555/your-org/vote-app-gitops

                cd vote-app-gitops/kubernetes
                git checkout -b feature-${BUILD_ID}

                sed -i "s#image: .*#image: ${env.DOCKER_IMAGE}#g" deployment.yml

                git add .
                git commit -m "Update vote-app image to ${env.GIT_COMMIT}"
                git push origin feature-${BUILD_ID}
                """
            }
        }

        stage('Raise PR for GitOps') {
            when {
                expression { env.BRANCH_NAME?.startsWith('PR') }
            }
            steps {
                withCredentials([string(credentialsId: 'gitea-token', variable: 'GITEA_TOKEN')]) {
                    sh """
                    curl -X POST \
                      -H "Authorization: token ${GITEA_TOKEN}" \
                      -H "Accept: application/json" \
                      -H "Content-Type: application/json" \
                      http://git-server:5555/api/v1/repos/your-org/vote-app-gitops/pulls \
                      -d '{
                        "title": "Update Docker Image ${env.GIT_COMMIT}",
                        "head": "feature-${BUILD_ID}",
                        "base": "main",
                        "body": "Automated PR for vote-app image update"
                      }'
                    """
                }
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
            echo "Cleaning workspace and publishing reports"

            junit allowEmptyResults: true, testResults: 'test-results.xml'

            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'coverage/lcov-report',
                reportFiles: 'index.html',
                reportName: 'Code Coverage HTML Report',
                useWrapperFileDirectly: true
            ])
        }

        failure {
            echo "Pipeline failed!"
        }

        success {
            echo "Pipeline completed successfully!"
        }
    }
}
