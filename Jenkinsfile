pipeline {
    // Use any agent
    agent any

    // Set the environment variable APP_PORT=9090
    environment {
        APP_PORT = '9090'
        // Optional Maven flags for non-interactive CI
        MAVEN_ARGS = '-B -U'
    }

    stages {
        stage('Build') {
            steps {
                // Use the maven package phase to build the project
                sh "mvn ${MAVEN_ARGS} -DskipTests clean package"
            }
        }
        stage('Unit Test') {
            steps {
                 // Use the maven test phase to run unit tests
                 sh "mvn ${MAVEN_ARGS} test"
            }
            post {
                always {
                    // Publish JUnit test results if present
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build & tests succeeded. APP_PORT=${env.APP_PORT}"
        }
        failure {
            echo "❌ Build or tests failed. Check logs and test reports."
        }
        always {
            // Archive artifacts if a JAR is produced
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, allowEmptyArchive: true
            cleanWs()
        }
    }
}
