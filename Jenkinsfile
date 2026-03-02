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
        IMAGE_DOCKERHUB = "nishantr/${APP_NAME}:latest"
        IMAGE_ECR = "306989527369.dkr.ecr.ap-south-1.amazonaws.com/demodockerrepo1/${APP_NAME}:latest"
        IMAGE_NEXUS = "13.232.59.26:8085/${APP_NAME}:latest"
        NEXUS_USERNAME = "admin"
        NEXUS_PASSWORD = "nexusadmin"
        DOCKERHUB_USER = "nishantr"
        DOCKERHUB_PASS = "nishant1"
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

/*
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
*/

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
                              docker login -u ${DOCKERHUB_USER} -p ${DOCKERHUB_PASS}
                              docker push ${IMAGE_DOCKERHUB}
                            """
                        }
                    }
                }

                stage('Push to Amazon ECR') {
                    steps {
                        withDockerRegistry(
                            credentialsId: 'ecr:ap-south-1:ecr-credentials',
                            url: "https://306989527369.dkr.ecr.ap-south-1.amazonaws.com"
                        ) {
                            sh "docker push ${IMAGE_ECR}"
                        }
                    }
                }

                stage('Push to Nexus') {
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                            sh """
                              docker login 13.232.59.26:8085 -u ${NEXUS_USERNAME} -p ${NEXUS_PASSWORD}
                              docker push ${IMAGE_NEXUS}
                            """
                        }
                    }
                }
            }
        }
        stage('Delete Docker Images from Jenkins Master') {
            steps {
                echo 'Cleaning Up Local Docker Images...'
                sh '''
                    docker rmi nishantr/bookmyplan:latest || echo "Image not found or already deleted"
                    docker rmi bookmyplan:latest || echo "Image not found or already deleted"
                    docker rmi 306989527369.dkr.ecr.ap-south-1.amazonaws.com/bookmyplan:latest || echo "Image not found or already deleted"
                    docker rmi 13.232.59.26:8085/bookmyplan:latest
                    docker image prune -f
                '''
                echo 'Local Docker Images Cleaned Up Successfully!'
            }
        }
    }
}
