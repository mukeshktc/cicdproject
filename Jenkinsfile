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
        NEXUS_PASS = 'admin'
        RELEASE_REPO = 'vpro-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUS_IP = '172.31.13.83'
        NEXUS_PORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN = 'nexuslogin'
        SONAR_SERVER = 'SONARSERVER'
        SONAR_SCANNER = 'SONARSCANNER'
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
        stage('UPDATE APPLICATION PROPERTIES'){
            steps{
                sh '''cat <<EOT> src/main/resources/application.properties
#JDBC Configutation for Database Connection
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://172.31.91.20:3306/accounts?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
jdbc.username=admin
jdbc.password=admin123

#Memcached Configuration For Active and StandBy Host
#For Active Host
memcached.active.host=172.31.91.20
memcached.active.port=11211
#For StandBy Host
memcached.standBy.host=127.0.0.2
memcached.standBy.port=11211

#RabbitMq Configuration
rabbitmq.address=172.31.91.20
rabbitmq.port=5672
rabbitmq.username=test
rabbitmq.password=test

#Elasticesearch Configuration
elasticsearch.host =192.168.1.85
elasticsearch.port =9300
elasticsearch.cluster=vprofile
elasticsearch.node=vprofilenode

EOT'''
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