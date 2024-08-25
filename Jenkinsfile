pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    stages {
        stage('Git Checkout') {
            steps {
                checkout scm: [
                    $class: 'GitSCM', 
                    branches: [[name: '*/master']],
                    extensions: [],
                    userRemoteConfigs: [[url: 'https://github.com/veera1016/java-kubernetes.git']]
                ]
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
                script {
                    try {
                        sh "trivy fs --scanners vuln --format table -o trivy-fs-report.html ."
                        archiveArtifacts artifacts: 'trivy-fs-report.html'
                    } catch (Exception e) {
                        error "Trivy File System Scan failed: ${e.message}"
                    }
                }
            }
        }
        stage('SonarQube Quality Gate') {
            steps {
                withSonarQubeEnv('SonarQubeServer') { // Replace 'SonarQubeServer' with your actual SonarQube server configuration name
                    try {
                        sh 'mvn sonar:sonar'
                    } catch (Exception e) {
                        error "SonarQube analysis failed: ${e.message}"
                    }
                }
                script {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
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
                    withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3') {
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
        stage('Docker Image Scan with Trivy') {
            steps {
                script {
                    try {
                        sh "trivy image --format table -o trivy-image-report.html veera1016/boardgame:latest"
                        archiveArtifacts artifacts: 'trivy-image-report.html'
                    } catch (Exception e) {
                        error "Trivy Docker Image Scan failed: ${e.message}"
                    }
                }
            }
        }
        stage('Push Docker Image to Registry') {
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
                withKubeConfig(
                    caCertificate: '', 
                    clusterName: 'kubernetes', 
                    contextName: '', 
                    credentialsId: 'k8-cred', 
                    namespace: 'webapps', 
                    restrictKubeConfigAccess: false, 
                    serverUrl: 'https://172.31.8.22:6443' // Replace with your Kubernetes API server URL
                ) {
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
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'
                def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h3 style="color: white; background-color:${bannerColor};">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
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
                    attachmentsPattern: 'trivy-fs-report.html,trivy-image-report.html'
                )
            }
        }
        failure {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def body = """
                <html>
                <body>
                <h3>Pipeline failed in job ${jobName} - Build ${buildNumber}</h3>
                <p>Check the <a href="${BUILD_URL}">console output</a> for details.</p>
                </body>
                </html>
                """
                emailext(
                    subject: "${jobName} - Build ${buildNumber} - FAILURE",
                    body: body,
                    to: 'togaruashok1996@gmail.com',
                    from: 'togaruashok1996@gmail.com',
                    replyTo: 'ashoktogaru1996@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
