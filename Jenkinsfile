pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Bijan1235/BoardGame.git'
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
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame \
-Dsonar.projectKey=BoardGame -Dsonar.java.binaries=.'''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        stage('Build and Deploy') {
            steps {
                // Use Jenkins credentials for Nexus
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
                    // Configure Maven with global settings, JDK, and Maven version
                    withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3') {
                        // Run Maven deploy to publish artifacts to Nexus
                        sh "mvn deploy"
                    }
                }
            }
        }
        stage('Build and Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t veera1016/boardgame:latest ."
                    }
                }
            }
        }
        stage('Docker Image Scan') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "trivy image --format table -o trivy-image-report.html veera1016/boardgame:latest"
                    }
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push veera1016/boardgame:latest"
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.8.22:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                    sh "kubectl get pods -n webapps"
                }
            }
        }
    }
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'
                def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </body>
                </html>
                """
                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'togaruashok1996@gmail.com',
                    from: 'togaruashok1996@gmail.com',
                    replyTo: 'ashoktogaru1996@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
 
