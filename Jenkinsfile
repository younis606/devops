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
