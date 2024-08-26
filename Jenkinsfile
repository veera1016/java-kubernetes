pipeline {
    agent any

    tools {
        jdk 'jdk17'                     // Ensure 'jdk17' is properly configured in Jenkins global tool configuration.
        maven 'Maven3'            // Ensure 'Default Maven' is properly configured in Jenkins global tool configuration.
    }

    environment {
        mvnHome = tool 'Maven3'   // Define the Maven tool path to be used in shell scripts.
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm              // Checks out the code from the SCM (e.g., Git).
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('sonarqubeserver') {   // 'sonarqubeserver' should match the name configured in Jenkins for SonarQube.
                        sh "${mvnHome}/bin/mvn clean verify sonar:sonar \
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
                    // waitForQualityGate abortPipeline: true   // Ensures pipeline stops if the quality gate fails.
                    echo "Quality gate check would be performed here."
                }
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'   // Scans the file system for vulnerabilities.
                archiveArtifacts artifacts: 'trivy-fs-report.html'       // Archives the Trivy file system scan report.
            }
        }

        stage('Docker Image Build and Scan') {
            steps {
                script {
                    withDockerRegistry(url: 'https://index.docker.io/v1/', credentialsId: 'docker-cred') {
                        sh 'docker build -t veera1016/boardgame:latest .'  // Builds the Docker image.
                        sh 'trivy image --format table -o trivy-image-report.html veera1016/boardgame:latest'  // Scans the Docker image.
                        archiveArtifacts artifacts: 'trivy-image-report.html'  // Archives the Trivy Docker image scan report.
                        sh 'docker push veera1016/boardgame:latest'        // Pushes the Docker image to Docker Hub.
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',           // 'k8-cred' should match the Jenkins credential ID for Kubernetes access.
                    serverUrl: 'https://172.31.8.22:6443'  // Replace with your actual Kubernetes API server URL.
                ) {
                    sh 'kubectl apply -f deployment-service.yaml'  // Deploys the application to Kubernetes.
                    sh 'kubectl get pods -n webapps'               // Lists the pods in the 'webapps' namespace.
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
