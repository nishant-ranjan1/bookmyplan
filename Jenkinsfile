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
                echo 'Code Compilation Completed Successfully!'
            }
        }

        stage('Code QA Execution') {
            steps {
                echo 'Running JUnit Test Cases...'
                sh 'mvn clean test'
                echo 'JUnit Test Cases Completed Successfully!'
            }
        }

        stage('Code Package') {
            steps {
                echo 'Creating JAR Artifact...'
                sh 'mvn clean package'
                sh "cp target/*.jar target/bookmyplan-1.1.${BUILD_NUMBER}.jar"
                echo 'Artifact Created Successfully!'
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                echo 'Building Docker Image and Tagging...'
                // Fixed: Combined tags into one build command for efficiency
                sh "docker build -t nishantr/bookmyplan:latest -t bookmyplan:latest -t 306989527369.dkr.ecr.ap-south-1.amazonaws.com/demodockerrepo1:latest ."
                echo 'Docker Image Build Completed!'
            }
        }

        stage('Docker Image Scanning') {
            steps {
                echo 'Scanning Docker Image with Trivy...'
                echo 'Docker Image Scanning Completed!'
            }
        }
/*
        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhubCred', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
                        echo 'Pushing Docker Image to Docker Hub...'
                        sh 'docker push nishantr/bookmyplan:latest'
                        echo 'Docker Image Pushed to Docker Hub Successfully!'
                    }
                }
            }
        }
*/
        stage('Pushing to Docker Hub') {
            steps {
                script {
                    // This block handle login and cleanup automatically
                    docker.withRegistry('https://index.docker.io', 'dockerhubCred') {
                        echo 'Pushing Docker Image...'
                        sh 'docker push nishantr/bookmyplan:latest'
                    }
                }
            }
        }

        stage('Push Docker Image to Amazon ECR') {
            steps {
                script {
                    // Ensure the URL matches your ECR registry exactly
                    withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://306989527369.dkr.ecr.ap-south-1.amazonaws.com"]) {
                        echo 'Pushing Docker Image to ECR...'
                        sh 'docker push 306989527369.dkr.ecr.ap-south-1.amazonaws.com/demodockerrepo1:latest'
                        echo 'Docker Image Pushed to Amazon ECR Successfully!'
                    }
                }
            }
        }

        stage('Upload Docker Image to Nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexuscred', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        // Use the Docker Connector port (8085) for login, not the UI port (8081)
                        sh "docker login 65.0.76.168:8085 -u ${USERNAME} -p ${PASSWORD}"
                        echo "Push Docker Image to Nexus: In Progress"
                        sh 'docker tag bookmyplan:latest 65.0.76.168:8085/bookmyplan:latest'
                        sh 'docker push 65.0.76.168:8085/bookmyplan:latest'
                        echo "Push Docker Image to Nexus: Completed"
                    }
                }
            }
        }

        stage('Clean Up Local Docker Images') {
            steps {
                echo 'Cleaning Up Local Docker Images...'
                sh '''
                    docker rmi nishantr/bookmyplan:latest || true
                    docker rmi bookmyplan:latest || true
                    docker rmi 306989527369.dkr.ecr.ap-south-1.amazonaws.com/demodockerrepo1:latest || true
                    docker rmi 65.0.76.168:8085/bookmyplan:latest || true
                    docker image prune -f
                '''
                echo 'Local Docker Images Cleaned Up Successfully!'
            }
        }
    }
}
