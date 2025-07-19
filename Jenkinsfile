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
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {
                        sh "docker build -t snehalatabarenkal/blogging-apps:latest ."
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
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {
                        sh 'docker push snehalatabarenkal/blogging-apps:latest'
                    }
                }
            }
        }

        stage('K8s Deployment') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8_cred',
                    clusterName: 'devopsshack-cluster',
                    serverUrl: 'https://C258CB2428552A92586882AB2A2D73D7.yl4.ap-northeast-3.eks.amazonaws.com',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false
                ) {
                    sh 'kubectl apply -f deployment-service.yml -n webapps'
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8_cred',
                    clusterName: 'devopsshack-cluster',
                    serverUrl: 'https://C258CB2428552A92586882AB2A2D73D7.yl4.ap-northeast-3.eks.amazonaws.com',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false
                ) {
                    sh 'kubectl get pods -n webapps'
                    sh 'kubectl get svc -n webapps'
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
                    to: 'laddubarenkal@gmail.com',
                    from: 'snehalatabarenkal2004@gmail.com',
                    replyTo: 'snehalatabarenkal2004@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
