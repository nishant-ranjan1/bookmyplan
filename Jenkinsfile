pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
    }

    tools {
        maven 'mvn_3.9.12'
    }

    stages {
        stage('Code Compilation') {
            steps {
                echo 'Starting Code Compilation...'
                sh 'mvn clean compile'
            }
        }

        stage('Code QA Execution') {
            steps {
                echo 'Running JUnit Test Cases...'
                sh 'mvn clean test'
            }
        }

        stage('SonarQube Code Quality') {
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

        stage('Code Package') {
            steps {
                echo 'Creating JAR Artifact...'
                sh 'mvn clean package'
                sh "cp target/*.jar target/bookmyplan-1.1.${BUILD_NUMBER}.jar"
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                echo 'Building Docker Image...'
                // Using --no-cache to fix the overlay2 error you encountered
                sh "docker build --no-cache -t nishantr/bookmyplan:latest -t bookmyplan:latest -t 306989527369.dkr.ecr.ap-south-1.amazonaws.com/demodockerrepo1:latest ."
            }
        }

        stage('Docker Image Scanning') {
            steps {
                echo 'Trivy Scanning is currently DISABLED in script.'
                // All sh commands are removed/commented correctly here
                // sh "trivy image ..."
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhubCred', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
                        sh 'docker push nishantr/bookmyplan:latest'
                    }
                }
            }
        }

        stage('Push Docker Image to Amazon ECR') {
            steps {
                script {
                    // Corrected closing braces for this block
                    withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://306989527369.dkr.ecr.ap-south-1.amazonaws.com"]) {
                        sh 'docker push 306989527369.dkr.ecr.ap-south-1.amazonaws.com/demodockerrepo1:latest'
                    }
                }
            }
        }

        stage('Clean Up Local Docker Images') {
            steps {
                sh '''
                    docker rmi nishantr/bookmyplan:latest || true
                    docker rmi bookmyplan:latest || true
                    docker rmi 306989527369.dkr.ecr.ap-south-1.amazonaws.com/demodockerrepo1:latest || true
                    docker image prune -f
                '''
            }
        }
    }
}
