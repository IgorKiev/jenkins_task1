pipeline {
    agent any

    tools {
        jdk 'JDK17'       // Use the exact names configured in Jenkins Global Tool Configuration
        maven 'Maven3'
    }

    environment {
        APP_PORT = '9090'
        MAVEN_OPTS = '-Dmaven.test.failure.ignore=false'
        MAVEN_CONFIG = '-B -U'
    }

    options {
        ansiColor('xterm')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 20, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                // Uses the SCM config from the job
                checkout scm
                sh 'git --no-pager log -1 --oneline || true'
            }
        }

        stage('Build') {
            steps {
                sh "mvn ${MAVEN_CONFIG} -DskipTests clean package"
            }
        }

        stage('Unit Tests') {
            steps {
                sh "mvn ${MAVEN_CONFIG} test"
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }

    post {
        success { echo "✅ Build successful!" }
        failure { echo "❌ Build failed!" }
        always   { cleanWs() }
    }
}
