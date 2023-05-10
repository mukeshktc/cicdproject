def COLOR_MAP = [
    'SUCCESS' : 'good',
    'FAILURE' : 'danger'
]

pipeline{
    agent any
    tools {
        maven 'MAVEN3'
        jdk 'OracleJDK8'
    }
    environment {
        SNAP_REPO = 'vpro-snapshots'
        NEXUS_USER = 'admin'
        NEXUS_PASS = credentials('nexuspass')
        RELEASE_REPO = 'vpro-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUS_IP = '172.31.13.83'
        NEXUS_PORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN = 'nexuslogin'
        SONAR_SERVER = 'SONARSERVER'
        SONAR_SCANNER = 'SONARSCANNER'
        DOCKER_PASS = credentials('dockerpass')
    }
    stages{
        stage('NOTIFICATION TO SLACK'){
            steps {
                echo 'Pipeline started'
            }
            post {
                always{
                    slackSend channel: '#devops-project',
                    color: 'good',
                    message: "Job is started Job name: ${env.JOB_NAME} build ${env.BUILD_NUMBER} time ${env.BUILD_TIMESTAMP} \n More info at: ${BUILD_URL}"
                }
            }
        }
        stage('BUILD'){
            steps{
                sh 'mvn -s settings.xml install -DskipTests'
            }
            post {
                success {
                    echo 'Now archiving'
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }
        stage('UNIT TEST'){
            steps{
                sh 'mvn -s settings.xml test'
            }
            post {
                success {
                    slackSend channel: '#devops-project',
                    color: 'good',
                    message: "UNIT TEST IS SUCCESS"
                }
                failure {
                    slackSend channel: '#devops-project',
                    color: 'danger',
                    message: "UNIT TEST IS FAILED"
                }
            }
        }
        stage('INTEGRATION TEST'){
            steps{
                sh 'mvn -s settings.xml verify -DskipUnitTests'
            }
        }
        stage('CHECKSTYLE ANALYSIS'){
            steps{
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
        }
        stage ('SONAR ANALYSIS') {
            environment {
                scannerHome = tool "${ SONAR_SCANNER}"
            }
            steps {
                withSonarQubeEnv("${SONAR_SERVER}") {
               sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
            }
            }
        }
        stage('QUALITY GATE') {
            steps{
                timeout (time:1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('UPLOAD ARTIFACT TO NEXUS'){
            steps{
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUS_IP}:${NEXUS_PORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: "${RELEASE_REPO}",
                    credentialsId: "${NEXUS_LOGIN}",
                    artifacts: [
                        [artifactId: 'vproapp',
                        classifier: '',
                        file: 'target/vprofile-v2.war',
                        type: 'war']
                    ]
                )
            }
        }
        stage('Ansible Deploy to Server'){
            steps{
                ansiblePlaybook([
                    inventory: 'ansible/inventory',
                    playbook: 'ansible/site.yml',
                    installation: 'ansible',
                    colorized: true,
                    credentialsId: 'applogin',
                    disableHostKeyChecking: true,
                    extraVars: [
                        USER: "${NEXUS_USER}",
                        PASS: "${NEXUS_PASS}",
                        nexusip: "${NEXUS_IP}",
                        reponame: "${RELEASE_REPO}",
                        groupid: 'QA',
                        artifactid: 'vproapp',
                        build: "${env.BUILD_ID}",
                        time: "${env.BUILD_TIMESTAMP}",
                        vprofile_version: "vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.war",
                        dockerPass: "${DOCKER_PASS}"
                    ]
                ])
            }
        }
        stage('Ansible Deploy to Kubernetes'){
            steps{
                ansiblePlaybook([
                    inventory: 'ansible/inventory',
                    playbook: 'ansible/kube.yml',
                    installation: 'ansible',
                    colorized: true,
                    credentialsId: 'kubelogin',
                    disableHostKeyChecking: true,
                ])
            }
        }
    }
    post {
        always {
            echo 'slack notifications'
            slackSend channel: '#devops-project',
            color: COLOR_MAP[currentBuild.currentResult],
            message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} time ${env.BUILD_TIMESTAMP} \n More info at: ${BUILD_URL}"
        }
    }
}