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
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/snehalatabarenkal/FullStack-Blogging-App.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=blogging-app \
                        -Dsonar.projectKey=blogging-app \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Publish Artifacts') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk17', maven: 'maven3') {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {
                        sh 'docker build -t snehalatabarenkal/blogging-apps:latest .'
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --format table -o image.html snehalatabarenkal/blogging-apps:latest'
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {
                        sh 'docker push snehalatabarenkal/blogging-apps:latest'
                    }
                }
            }
        }

        stage('K8s Deploy App') {
            steps {
                withKubeConfig(
                    caCertificate: '',
                    clusterName: 'devopsshack-cluster',
                    contextName: '',
                    credentialsId: 'k8_cred',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://4F4FAFBBDEE98E54783E6B4734B4F404.yl4.ap-northeast-3.eks.amazonaws.com'
                ) {
                    sh "kubectl apply -f deployment-service.yml -n webapps"
                }
            }
        }

        stage('K8s Status Check') {
            steps {
                withKubeConfig(
                    caCertificate: '',
                    clusterName: 'devopsshack-cluster',
                    contextName: '',
                    credentialsId: 'k8_cred',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://4F4FAFBBDEE98E54783E6B4734B4F404.yl4.ap-northeast-3.eks.amazonaws.com'
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
