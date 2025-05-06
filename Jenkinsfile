pipeline {
    agent { label 'jenkins-Agent' }

    environment {
        APP_NAME = "frontBankingDEVOPS"
        RELEASE = "1.0.0"
        DOCKER_IMAGE = "rima603/frontbankadevops"
        SONARQUBE_ENV = "sonarqube"
        SONAR_PROJECT_KEY = "frontbanking"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git(
                    branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/rima-gif/frontBanka.git'
                )
            }
        }

        stage("Install Dependencies") {
            steps {
                sh 'npm install'
            }
        }

        stage("Build Angular App") {
            steps {
                sh 'npm run build'
                archiveArtifacts artifacts: 'dist/**', fingerprint: true
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    withCredentials([string(credentialsId: 'jenkins-sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=$SONAR_HOST_URL \
                              -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage("Wait for Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Build Docker Image") {
            steps {
                script {
                    sh """
                        docker build -t ${DOCKER_IMAGE}:${RELEASE} .
                        docker tag ${DOCKER_IMAGE}:${RELEASE} ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage("Trivy Security Scan") {
            steps {
                sh """
                    export TRIVY_TIMEOUT=10m
                    trivy image ${DOCKER_IMAGE}:${RELEASE} || true
                """
            }
        }

        stage("Push Docker Image") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${DOCKER_IMAGE}:${RELEASE}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
