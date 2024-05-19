
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Pramod858/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build"){
            steps{
                script{
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                        sh "docker build --build-arg TMDB_V3_API_KEY=<TMDB_API_KEY> -t netflix ."
                        sh "docker tag netflix pramod858/netflix:v${BUILD_NUMBER}"
                    }
                }
            }
        }
        stage("TRIVY Docker Scan"){
            steps{
                sh "trivy image pramod858/netflix:v${BUILD_NUMBER} > trivyimage.txt" 
            }
        }
        stage("Docker Push"){
            steps{
                script{
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                        sh "docker push pramod858/netflix:v${BUILD_NUMBER}"
                    }
                }
            }
        }
        stage('Deploy to container'){
            steps{
                sh "docker run -d --name netflix -p 8081:80 pramod858/netflix:v${BUILD_NUMBER}"
            }
        }
        stage('Update GIT') {
            steps{
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withCredentials([usernamePassword(credentialsId: 'git-cred', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        //def encodedPassword = URLEncoder.encode("$GIT_PASSWORD",'UTF-8')
                        sh "git config user.email pramodbadiger45@gmail.com"
                        sh "git config user.name Pramod858"
                        //sh "git switch master"
                        sh "cat K8s-Manifest/deployment-service.yaml"
                        sh "sed -i 's+pramod858/netflix.*+pramod858/netflix:v${BUILD_NUMBER}+g' K8s-Manifest/deployment-service.yaml"
                        sh "cat K8s-Manifest/deployment-service.yaml"
                        sh "git add K8s-Manifest/deployment-service.yaml"
                        sh "git commit -m 'Done by Jenkins Job changemanifest:v${env.BUILD_NUMBER}'"
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/DevSecOps-Project.git HEAD:main"
                    }
                }
            }
        }
    }
    post {
        success {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                
                def successfulHTML = """
                    <!DOCTYPE html>
                    <html>
                    <head>
                        <title>Pipeline Successful</title>
                    </head>
                    <body style="font-family: Arial, sans-serif;">
                        <div style="background-color: #DFF0D8; padding: 20px;">
                            <h2 style="color: #3C763D;">Pipeline Successful</h2>
                            <p>Your pipeline for ${jobName} build ${buildNumber} has succeeded. Great job!</p>
                            <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                        </div>
                    </body>
                    </html>
                """
                emailext subject: "Pipeline Successful",
                            body: successfulHTML,
                            from: "jenkins@example.com",
                            to: "recipientemail",
                            replyTo: "jenkins@example.com",
                            mimeType: 'text/html'
            }
        }
        failure {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                
                def failedHTML = """
                    <!DOCTYPE html>
                    <html>
                    <head>
                        <title>Pipeline Failed</title>
                    </head>
                    <body style="font-family: Arial, sans-serif;">
                        <div style="background-color: #F2DEDE; padding: 20px;">
                            <h2 style="color: #A94442;">Pipeline Failed</h2>
                            <p>Your pipeline for ${jobName} build #${buildNumber} has failed. Please check and take necessary actions.</p>
                            <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                        </div>
                    </body>
                    </html>
                """
                emailext subject: "Pipeline Failed",
                            body: failedHTML,
                            from: "jenkins@example.com",
                            to: "recipientemail",
                            replyTo: "jenkins@example.com",
                            mimeType: 'text/html'
            }
        }
    }
}


