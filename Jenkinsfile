
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
                git branch: 'main', url: 'https://github.com/KenMutuma/DevSecOps-Project.git'
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
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
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
   
        stage('Checkout') {
             steps {
                 git branch: 'master', url: 'https://github.com/bridgecrewio/terragoat'
                 stash includes: '**/*', name: 'terragoat'
             }
         }
         stage('Checkov Security Scan') {
             steps {
                 script {
                     docker.image('bridgecrew/checkov:latest').inside("--entrypoint=''") {
                         unstash 'terragoat'
                         try {
                             sh 'checkov -d . --framework kubernetes -o json --output-file-path checkov_results.json --repo-id example/terragoat --branch master --quiet'
                             junit allowEmptyResults: true, testResults: 'checkov_results.json'
                             archiveArtifacts artifacts: 'checkov_results.json', fingerprint: true
                         } catch (err) {
                             junit allowEmptyResults: true, testResults: 'checkov_results.json'
                             archiveArtifacts artifacts: 'checkov_results.json', fingerprint: true
                             error "Checkov found critical vulnerabilities! View report in artifacts."
                         }
                     }
                 }
              }
         }
     
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=71dcc83fe147e36961775288e97875b1 -t netflix ."
                       sh "docker tag netflix rubbicon/netflix:latest "
                       sh "docker push rubbicon/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image rubbicon/netflix:latest > trivyimage.txt" 
            }
        }
        // stage('Deploy to container'){
        //     steps{
        //         sh 'docker run -d -p 8081:80 rubbicon/netflix:latest'
        //     }
        // }
    }
    options {
         preserveStashes()
         timestamps()
    }
 post {
    always {
        emailext(
            subject: "'${currentBuild.result}' - Job: ${env.JOB_NAME} (Build #${env.BUILD_NUMBER})",
            body: """
                <p><b>Build Status:</b> ${currentBuild.result}</p>
                <p><b>Project:</b> ${env.JOB_NAME}</p>
                <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
            """,
            to: 'kennethsila2@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt,checkov.json',
            attachLog: true
         )
         }
    }
}  


