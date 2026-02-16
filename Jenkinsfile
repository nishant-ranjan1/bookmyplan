pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
        timestamps()
    }

    tools {
        maven 'mvn_3.9.12'
    }

    environment {
        APP_NAME = "bookmyplan"
        VERSION  = "1.1.${BUILD_NUMBER}"
        IMAGE_LOCAL = "${APP_NAME}:latest"
        IMAGE_DOCKERHUB = "satyam88/${APP_NAME}:latest"
        IMAGE_ECR = "445842764710.dkr.ecr.ap-south-1.amazonaws.com/${APP_NAME}:latest"
        IMAGE_NEXUS = "3.108.228.196:8085/${APP_NAME}:latest"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                echo 'Building and running tests...'
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Code Quality') {
            environment {
                scannerHome = tool 'qube'
            }
            steps {
                echo 'Running SonarQube scan...'
                withSonarQubeEnv('sonar-server') {
                    sh 'mvn sonar:sonar'
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package Artifact') {
            steps {
                echo 'Packaging application...'
                sh "mvn package -DskipTests"
                sh "cp target/*.jar target/${APP_NAME}-${VERSION}.jar"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh """
                  docker build -t ${IMAGE_LOCAL} .
                  docker tag ${IMAGE_LOCAL} ${IMAGE_DOCKERHUB}
                  docker tag ${IMAGE_LOCAL} ${IMAGE_ECR}
                  docker tag ${IMAGE_LOCAL} ${IMAGE_NEXUS}
                """
            }
        }

        stage('Scan Docker Image') {
            steps {
                echo 'Scanning image with Trivy...'
                sh "trivy image ${IMAGE_LOCAL} || echo '⚠️ Trivy found issues, continuing...'"
            }
        }

        stage('Push Images') {
            parallel {

                stage('Push to Docker Hub') {
                    steps {
                        withCredentials([string(credentialsId: 'dockerhubCred', variable: 'DOCKERHUB_PASS')]) {
                            sh """
                              docker login -u satyam88 -p ${DOCKERHUB_PASS}
                              docker push ${IMAGE_DOCKERHUB}
                            """
                        }
                    }
                }

                stage('Push to Amazon ECR') {
                    steps {
                        withDockerRegistry(
                            credentialsId: 'ecr:ap-south-1:ecr-credentials',
                            url: "https://445842764710.dkr.ecr.ap-south-1.amazonaws.com"
                        ) {
                            sh "docker push ${IMAGE_ECR}"
                        }
                    }
                }

                stage('Push to Nexus') {
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                            sh """
                              docker login 3.108.228.196:8085 -u ${USERNAME} -p ${PASSWORD}
                              docker push ${IMAGE_NEXUS}
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up local Docker images...'
            sh """
              docker rmi ${IMAGE_LOCAL} || true
              docker rmi ${IMAGE_DOCKERHUB} || true
              docker rmi ${IMAGE_ECR} || true
              docker rmi ${IMAGE_NEXUS} || true
              docker image prune -f || true
            """
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Please check logs."
        }
    }
}
