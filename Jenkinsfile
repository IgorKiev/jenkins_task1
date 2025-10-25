pipeline {
    agent any

    environment {
        APP_PORT = '9090'
    }

    stages {
        stage('Info') {
            steps {
                echo "Pipeline started. APP_PORT=${env.APP_PORT}"
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully."
        }
        failure {
            echo "❌ Pipeline failed."
        }
        always {
            cleanWs()
        }
    }
}
