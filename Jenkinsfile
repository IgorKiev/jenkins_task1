pipeline {
  agent any

  /*
   * Make sure Jenkins has these tool installations configured:
   *  - JDK named 'JDK17' (or change to the one you have)
   *  - Maven named 'Maven3' (or change accordingly)
   */
  tools {
    jdk 'JDK17'
    maven 'Maven3'
  }

  // Environment variables for the pipeline and the application
  environment {
    // The app will read this to run on port 9090 (if coded to respect APP_PORT)
    APP_PORT = '9090'

    // Typical Maven options for consistent builds in CI
    MAVEN_OPTS = '-Dmaven.test.failure.ignore=false'
    // Improve Maven performance in CI (optional)
    MAVEN_CONFIG = '-B -U'
  }

  options {
    ansiColor('xterm')
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
    // Fail the build if any stage is left idle for too long
    timeout(time: 20, unit: 'MINUTES')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          echo "Branch: ${env.BRANCH_NAME ?: 'N/A'}"
        }
      }
    }

    stage('Validate Tooling') {
      steps {
        sh '''
          echo "Java version:"
          java -version
          echo "Maven version:"
          mvn -v
        '''
      }
    }

    stage('Build') {
      steps {
        // Clean and compile/package without running tests here
        sh """
          mvn ${MAVEN_CONFIG} -DskipTests clean package
        """
      }
    }

    stage('Unit Tests') {
      steps {
        // Run unit tests and produce reports
        sh """
          mvn ${MAVEN_CONFIG} test
        """
      }
      post {
        always {
          // Publish Surefire/JUnit results even if tests fail
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
        }
      }
    }

    // Optional: quick smoke test if your app produces a runnable JAR named target/*.jar
    // You can safely remove this stage if not needed.
    stage('Smoke Test (optional)') {
      when {
        expression {
          // Only attempt if a JAR exists after build
          def jars = sh(script: "ls -1 target/*.jar 2>/dev/null | wc -l", returnStdout: true).trim()
          return jars.toInteger() > 0
        }
      }
      steps {
        script {
          // Find the first jar in target/
          def jarPath = sh(script: "ls -1 target/*.jar | head -n 1", returnStdout: true).trim()
          echo "Starting app from ${jarPath} on port ${env.APP_PORT} for a short smoke check..."

          // Start app in the background with the APP_PORT env available
          sh """
            set -e
            export APP_PORT=${APP_PORT}
            nohup java -jar "${jarPath}" >/tmp/app.log 2>&1 &
            echo \$! > app.pid
          """

          // Give the app some time to start (tune as needed)
          sleep 8

          // If your app exposes a health endpoint, adjust the URL below.
          // We'll just check that the port is listening to avoid hard-coding a path.
          sh """
            if command -v nc >/dev/null 2>&1; then
              nc -z localhost ${APP_PORT}
            else
              # Fallback: try curl loop to the root path (update if your app needs a specific path)
              for i in {1..5}; do
                curl -sSf "http://localhost:${APP_PORT}/" >/dev/null && exit 0 || sleep 2
              done
              exit 1
            fi
          """
        }
      }
      post {
        always {
          // Stop the app if it is running
          sh '''
            if [ -f app.pid ]; then
              kill $(cat app.pid) 2>/dev/null || true
              rm -f app.pid
            fi
          '''
          // Archive logs for investigation
          archiveArtifacts artifacts: 'target/*.jar, /tmp/app.log', allowEmptyArchive: true
        }
      }
    }

    stage('Archive Artifacts') {
      steps {
        // Keep JARs produced by Maven for downstream or download
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, allowEmptyArchive: true
      }
    }
  }

  post {
    success {
      echo "✅ Build successful on branch ${env.BRANCH_NAME ?: 'N/A'}."
    }
    failure {
      echo "❌ Build failed. Check the console log and test reports."
    }
    always {
      // Clean workspace to avoid cross-build pollution
      cleanWs()
    }
  }
}
