pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    stages {
        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        stage('Trivy File System Scan') {
            steps {
                sh "trivy fs --scanners vuln --format table -o trivy-fs-report.html ."
                archiveArtifacts artifacts: 'trivy-fs-report.html'
            }
        }
        stage('SonarQube Quality Gate') {
            steps {
                withSonarQubeEnv('SonarQubeServer') { // Replace with your actual SonarQube server configuration
                    sh 'mvn sonar:sonar'
                }
                script {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Quality Gate failed: ${qg.status}"
                    }
                }
            }
        }
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        stage('Deploy to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
                    sh "mvn deploy"
                }
            }
        }
        stage('Build and Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t veera1016/boardgame:latest ."
                    }
                }
            }
        }
        stage('Docker Image Scan with Trivy') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html veera1016/boardgame:latest"
                archiveArtifacts artifacts: 'trivy-image-report.html'
            }
        }
        stage('Push Docker Image to Registry') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred') {
                    sh "docker push veera1016/boardgame:latest"
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    serverUrl: 'https://172.31.8.22:6443' // Replace with your Kubernetes API server URL
                ) {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }
    }
    post {
        always {
            script {
                def status = currentBuild.result ?: 'SUCCESS'
                emailext(
                    subject: "Build ${env.BUILD_NUMBER} - ${status}",
                    body: "Check the <a href='${env.BUILD_URL}'>console output</a> for details.",
                    to: 'togaruashok1996@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
