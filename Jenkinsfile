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
    }
    stages{
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
        stage('TEST'){
            steps{
                sh 'mvn -s settings.xml test'
            }
        }
        stage('CHECKSTYLE ANALYSIS'){
            steps{
                sh 'mvn -s settings.xml checksyle:checkstyle'
            }
        }

    }
}