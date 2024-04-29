pipeline {
    agent any 

    tools {
        jdk "jdk17"
        maven "maven3"
    }

    environment {
        SCANNER_HOME=tool "sonar-scanner"
    }

    stages{
        stage("Git Checkout") {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url:"https://github.com/Pawan-choudhary/Board-Game.git", 
            }
        }

        stage("Compile") {
            staps {
                sh "mvn compile"
            }
        }

        stage("Test") {
            staps {
                sh "mvn test"
            }
        }

        stage("File System Scan") {
            staps {
                sh "trivy fs --formate table -o trivy-fs-report.html ."
            }
        }

        stage("SonarQube Analysis") {
            staps {
                withSonarQubeEnv('sonar-server')
                sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
	                    -Dsonar.java.binaries=. '''
            }
        }

        stage("Quality Gate") {
            staps {
                script {
                    waitForQualityGate(abortPipeline: false, credentialsId: 'sonar-token')
                }
            }
        }

        stage("Build") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Publish to Nexus") {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk:"jdk17", maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }

        stage("Build and tag docker image") {
            steps {
                script {
                    withDockerRegistry(credentialsID: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t pawanchoudharynain/boarshack:latest ."
                    }
                }
            }
        }

        stage("Dcoker Image Scan") {
            steps {
                sh "trivy image --formate table -o trivy-image-report.html pawanchoudharynain/boarshack:latest"
            }
        }

        stage("Push Docker Image") {
            steps {
                script {
                    withDockerRegistry(credentialsID: 'docker-cred', toolName: 'docker') {
                        sh "docker push pawanchoudharynain/boarshack:latest"
                    }
                }
            }
        }

        stage("Deploy to k8S") {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kunernetes',  contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess:false, serverUrl: '') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage("Verify the deployment") {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kunernetes',  contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess:false, serverUrl: '') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
        // Setting up E-mail Notification
        stage("E-mail Notification") {
            steps {
                sh "mvn clean package"
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
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'jpawanchoudhary27437@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}
}