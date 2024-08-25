pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/veera1016/java-kubernetes.git', branch: 'master'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
                archiveArtifacts artifacts: 'trivy-fs-report.html'
            }
        }
        stage('SonarQube Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        stage('Deploy to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh "mvn deploy -DaltDeploymentRepository=nexus::default::http://nexus-url/repository/maven-releases/"
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t veera1016/boardgame:latest .'
                }
            }
        }
        stage('Docker Image Scan with Trivy') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html veera1016/boardgame:latest'
                archiveArtifacts artifacts: 'trivy-image-report.html'
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker push veera1016/boardgame:latest'
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred') {
                    sh 'kubectl apply -f deployment-service.yaml'
                    sh 'kubectl get pods -n webapps'
                }
            }
        }
    }
    post {
        always {
            emailext(
                subject: "Build ${currentBuild.fullDisplayName} - ${currentBuild.result ?: 'SUCCESS'}",
                body: """<p>${currentBuild.fullDisplayName} completed with status: ${currentBuild.result ?: 'SUCCESS'}</p>
                        <p>Check console output at ${env.BUILD_URL}</p>""",
                to: 'togaruashok1996@gmail.com',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}
