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

        stage('SonarQube Code Quality') {
             environment {
                 scannerHome = tool 'qube'
             }
             steps {
                 echo 'Starting SonarQube Code Quality Scan...'
                 withSonarQubeEnv('sonar-server') {
                     sh 'mvn sonar:sonar'
                 }
                 echo 'SonarQube Scan Completed. Checking Quality Gate...'
                 timeout(time: 10, unit: 'MINUTES') {
                     waitForQualityGate abortPipeline: true
                 }
                 echo 'Quality Gate Check Completed!'
             }
        }

        stage('Code Package') {
            steps {
                echo 'Creating WAR Artifact...'
                sh 'mvn clean package'
                sh '''
                    cp target/*.jar target/bookmyplan-1.1.${BUILD_NUMBER}.jar
                '''
                echo 'WAR Artifact Created Successfully!'
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                echo 'Building Docker Image and Tagging...'
                sh "docker build -t satyam88/bookmyplan:latest -t bookmyplan:latest ."
                echo 'Docker Image Build Completed!'
            }
        }

        stage('Docker Image Scanning') {
            steps {
                echo 'Scanning Docker Image with Trivy...'
                echo 'Docker Image Scanning Completed!'
            }
        }
        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'dockerhubCred', variable: 'dockerhubCred')]) {
                        sh 'docker login docker.io -u satyam88 -p ${dockerhubCred}'
                        echo 'Pushing Docker Image to Docker Hub...'
                        sh 'docker push satyam88/bookmyplan:latest'
                        echo 'Docker Image Pushed to Docker Hub Successfully!'
                    }
                }
            }
        }
        stage('Push Docker Image to Amazon ECR') {
            steps {
                script {
                    withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://445842764710.dkr.ecr.ap-south-1.amazonaws.com"]) {
                        echo 'Tagging and Pushing Docker Image to ECR...'
                        sh '''
                            docker images
                            docker tag bookmyplan:latest 445842764710.dkr.ecr.ap-south-1.amazonaws.com/bookmyplan:latest
                            docker push 445842764710.dkr.ecr.ap-south-1.amazonaws.com/bookmyplan:latest
                        '''
                        echo 'Docker Image Pushed to Amazon ECR Successfully!'
                    }
                }
            }
        }
        stage('Upload Docker Image to Nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh 'docker login http://3.108.228.196:8085/repository/bookmyplan/ -u admin -p ${PASSWORD}'
                        echo "Push Docker Image to Nexus : In Progress"
                        sh 'docker tag bookmyplan 3.108.228.196:8085/bookmyplan:latest'
                        sh 'docker push 3.108.228.196:8085/bookmyplan'
                        echo "Push Docker Image to Nexus : Completed"
                    }
                }
            }
        }
        stage('Clean Up Local Docker Images') {
            steps {
                echo 'Cleaning Up Local Docker Images...'
                sh '''
                    docker rmi satyam88/bookmyplan:latest || echo "Image not found or already deleted"
                    docker rmi bookmyplan:latest || echo "Image not found or already deleted"
                    docker rmi 445842764710.dkr.ecr.ap-south-1.amazonaws.com/bookmyplan:latest || echo "Image not found or already deleted"
                    docker rmi 65.0.76.100:8085/bookmyplan:latest
                    docker image prune -f
                '''
                echo 'Local Docker Images Cleaned Up Successfully!'
            }
        }
    }
}
