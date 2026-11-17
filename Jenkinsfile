
pipeline {
    agent any

    tools {
        maven 'Maven3'
        
    }

    environment {
        DOCKERHUB_CRED_ID = 'dockerhub-credentials'
        DOCKERHUB_REPO = 'swathipeddakota/simple-docker-webapp'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        CONTAINER_NAME = 'simple-docker-webapp'
        APP_PORT = '8080'
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/swathi-peddakota/simple-docker-webapp.git'
            }
        }

        stage('Build & Test (Maven)') {
            steps {
                sh 'mvn -B clean verify'
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml',
                        allowEmptyResults: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarQube1') {
                    sh '''
                        mvn -B sonar:sonar \
                        -Dsonar.projectKey=simple-docker-webapp \
                        -Dsonar.projectName=simple-docker-webapp
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                    -t ${DOCKERHUB_REPO}:${IMAGE_TAG} \
                    -t ${DOCKERHUB_REPO}:latest .
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: DOCKERHUB_CRED_ID,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        printf '%s' "$DOCKER_PASS" | docker login \
                        -u "$DOCKER_USER" --password-stdin

                        docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                        docker push ${DOCKERHUB_REPO}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    docker pull ${DOCKERHUB_REPO}:latest

                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true

                    docker run -d \
                    --name ${CONTAINER_NAME} \
                    -p ${APP_PORT}:8080 \
                    --restart unless-stopped \
                    ${DOCKERHUB_REPO}:latest
                """
            }
        }

        stage('Health Check') {
            steps {
                sh """
                    for i in \$(seq 1 10); do
                        if curl -sf http://localhost:${APP_PORT}/actuator/health; then
                            echo 'App is healthy'
                            exit 0
                        fi

                        echo 'Waiting for app to become healthy...'
                        sleep 5
                    done

                    echo 'Health check failed after retries'
                    exit 1
                """
            }
        }
    }

    post {
        success {
            echo "Successfully built, pushed and deployed ${DOCKERHUB_REPO}:${IMAGE_TAG}"
        }

        failure {
            echo 'Pipeline failed - check console output above.'
        }

        always {
            sh 'docker image prune -f || true'
        }
    }
}
