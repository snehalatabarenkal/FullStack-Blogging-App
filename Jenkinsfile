pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/snehalatabarenkal/FullStack-Blogging-App.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean install'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                sh 'rm -rf .scannerwork'
                withSonarQubeEnv('SonarQube') {
                    sh '''${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=blogging1 \
                        -Dsonar.projectName=blogging1 \
                        -Dsonar.projectVersion=${BUILD_NUMBER} \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.java.binaries=target'''
                }
            }
        }

        stage('Artifact Publish') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    def tag = "${env.BUILD_NUMBER}"
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {
                        sh """
                            docker build -t snehalatabarenkal/blogging-apps:${tag} .
                            docker tag snehalatabarenkal/blogging-apps:${tag} snehalatabarenkal/blogging-apps:latest
                        """
                    }
                }
            }
        }

        stage('Scan Docker Image by Trivy') {
            steps {
                sh 'trivy image --format table -o image-report.html snehalatabarenkal/blogging-apps:latest'
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    def tag = "${env.BUILD_NUMBER}"
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {
                        sh """
                            docker push snehalatabarenkal/blogging-apps:${tag}
                            docker push snehalatabarenkal/blogging-apps:latest
                        """
                    }
                }
            }
        }

        stage('K8s Deployment') {
            steps {
                withKubeCredentials(kubectlCredentials: [[
                    caCertificate: '', 
                    clusterName: 'devopsshack-cluster', 
                    contextName: '', 
                    credentialsId: 'k8_cred', 
                    namespace: 'webapps', 
                    serverUrl: 'https://D36CD53F592EAF67D8C3F2552340B3E6.sk1.ap-northeast-3.eks.amazonaws.com'
                ]]) {
                    sh 'kubectl apply -f deployment-service.yml'
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeCredentials(kubectlCredentials: [[
                    caCertificate: '', 
                    clusterName: 'devopsshack-cluster', 
                    contextName: '', 
                    credentialsId: 'k8_cred', 
                    namespace: 'webapps', 
                    serverUrl: 'https://D36CD53F592EAF67D8C3F2552340B3E6.sk1.ap-northeast-3.eks.amazonaws.com'
                ]]) {
                    sh 'kubectl get pods'
                    sh 'kubectl get svc'
                }
            }
        }
    }

    post {
        always {
            script {
                def pipelineStatus = currentBuild.result == null ? 'SUCCESS' : currentBuild.result
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'
                def body = """
                    <html>
                        <body>
                            <div style="border: 4px solid #000; padding: 10px;">
                                <b style="background-color: ${bannerColor}; color: white; padding: 10px;">
                                    Pipeline Status: ${pipelineStatus.toUpperCase()}
                                </b>
                                <br><br>
                                Check the full log at: 
                                <a href="${env.BUILD_URL}console">${env.JOB_NAME} #${env.BUILD_NUMBER}</a>
                            </div>
                        </body>
                    </html>
                """

                emailext(
                    subject: "Jenkins Job: ${env.JOB_NAME} #${env.BUILD_NUMBER} - ${pipelineStatus}",
                    body: body,
                    to: 'amrutaadma20@gmail.com ',
                    from: 'snehalatabarenkal2004@gmail.com',
                    replyTo: 'snehalatabarenkal2004@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
