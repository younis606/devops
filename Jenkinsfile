pipeline {
    agent any

    environment {
        REGISTRY = "younis606"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Services with Docker Compose') {
            steps {
                echo "Building all services using docker-compose..."
                sh "docker-compose -f docker-compose.yml build"
            }
        }

        stage('Push Images to Docker Hub') {
            when {
                expression { return env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master' }
            }
            steps {
                script {
                    sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"

                    def images = sh(
                        script: "docker-compose config --services",
                        returnStdout: true
                    ).trim().split('\n')

                    images.each { svc ->
                        sh """
                            IMAGE=${REGISTRY}/${svc}:latest
                            docker tag ${svc}:latest \$IMAGE
                            docker push \$IMAGE
                        """
                    }
                }
            }
        }

        stage('Run Services') {
            steps {
                echo "Running all services using docker-compose..."
                sh "docker-compose -f docker-compose.yml up -d"
            }
        }

        stage('Security Scan (Trivy)') {
            steps {
                script {
                    def images = sh(
                        script: "docker-compose config --services",
                        returnStdout: true
                    ).trim().split('\n')

                    images.each { svc ->
                        sh "trivy image ${svc}:latest || true"
                    }
                }
            }
        }
    }
}
