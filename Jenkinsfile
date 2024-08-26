pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'Default Maven'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('sonarqubeserver') {
                        sh "${mvn}/bin/mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=boardgame \
                            -Dsonar.projectName='boardgame'"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // Uncomment the following line to enable quality gate check
                    // waitForQualityGate abortPipeline: true, credentialsId: 'sonar-cred'
                    echo "Quality gate check would be performed here."
                }
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
                archiveArtifacts artifacts: 'trivy-fs-report.html'
            }
        }

        stage('Docker Image Build and Scan') {
            steps {
                script {
                    withDockerRegistry(url: 'https://index.docker.io/v1/', credentialsId: 'docker-cred') {
                        sh 'docker build -t veera1016/boardgame:latest .'
                        sh 'trivy image --format table -o trivy-image-report.html veera1016/boardgame:latest'
                        archiveArtifacts artifacts: 'trivy-image-report.html'
                        sh 'docker push veera1016/boardgame:latest'
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    serverUrl: 'https://172.31.8.22:6443'
                ) {
                    sh 'kubectl apply -f deployment-service.yaml'
                    sh 'kubectl get pods -n webapps'
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def status = currentBuild.result ?: 'SUCCESS'
                def color = status == 'SUCCESS' ? 'green' : 'red'
                def body = """
                <html>
                <body>
                <div style="border: 4px solid ${color}; padding: 10px;">
                <h3 style="color: white; background-color: ${color};">Pipeline Status: ${status}</h3>
                </div>
                <p>Check the <a href="${env.BUILD_URL}">console output</a>.</p>
                </body>
                </html>
                """
                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${status}",
                    body: body,
                    to: 'togaruashok1996@gmail.com',
                    from: 'togaruashok1996@gmail.com',
                    replyTo: 'ashoktogaru1996@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-fs-report.html,trivy-image-report.html'
                )
            }
        }
    }
}
