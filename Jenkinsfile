pipeline {
    agent any

    environment {
        APP_PORT = '9090'
        MAVEN_ARGS = '-B -U'
    }

    stages {
        stage('Unit Test') {
            steps {
                // Run Maven tests
                sh "mvn ${MAVEN_ARGS} test"
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }
    }

    post {
        success {
            echo "✅ Tests completed successfully. APP_PORT=${env.APP_PORT}"
        }
        failure {
            echo "❌ Tests failed. Check logs and reports."
        }
        always {
            cleanWs()
        }
    }
}
