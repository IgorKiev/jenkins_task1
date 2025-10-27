pipeline {
    agent any
    environment {
        APP_PORT = '9090'
    }
    stages {
        stage('Build') {
            steps {
              
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Unit Test') {
            steps {
               
                sh 'mvn test'
            }
        }
    }
}
