pipeline {

    agent any
    environment {
        REGISTERY ="parimal5/kube-app"
        REGISTERY_CREDENTIALS= "dockerhub" //Credential in Jenkins Id- dockerhub (docker username and password)
    }

    // Docker is installed in Jenkins node
    // Helm is installed in kops
    // So we can directely used Docker but for helm we are using the slave node to trigger the job

    stages{
        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }
        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('BUILD App Image'){
            steps{
                script{
                    dockerImage = docker.build REGISTERY + ":V$BUILD_NUMBER"
                }
            }
        }

        stage('Upload Image'){
            steps{
                script{
                    // docker.withRegistry(URL, CRED) if not url mentioned it will use default (dockerhub)
                    docker.withRegistry('', REGISTERY_CREDENTIALS){
                        dockerImage.push("V$BUILD_NUMBER")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Delete Old Version'){
            steps{
                sh "docker rmi $REGISTERY:V$BUILD_NUMBER"
            }
        }

        stage('Kubernetes deploy'){
            agent{label 'KOPS'}
            steps{
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${REGISTERY}:V${BUILD_NUMBER} --namespace prod"
            }
        }

    }   

}
