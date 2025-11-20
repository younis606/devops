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
                sh 'npm test'
            }
        }

        stage('Code Coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE', message: 'Coverage issues') {
                    sh 'npm run coverage'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${env.DOCKER_IMAGE} ."
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    sh """
                    trivy image ${env.DOCKER_IMAGE} \
                      --severity CRITICAL,HIGH \
                      --exit-code 1 \
                      --format json -o trivy-image-results.json
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry([credentialsId: 'docker-hub-credentials', url: "http://${DOCKER_REGISTRY}"]) {
                        sh "docker push ${env.DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy via Helm (Dev)') {
            steps {
                script {
                    sh """
                    helm upgrade --install ${HELM_RELEASE} ${HELM_CHART_PATH} \
                      --namespace vote --create-namespace \
                      --kubeconfig ${KUBECONFIG_DEV} \
                      --values ${HELM_CHART_PATH}/values.yaml
                    """
                }
            }
        }

        stage('Smoke Test') {
            steps {
                script {
                    sh """
                    HTTP_STATUS=\$(curl_
