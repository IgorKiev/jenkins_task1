pipeline {
    agent any

    environment {
        APP_PORT   = '9090'
        MAVEN_ARGS = '-B -U'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
        timeout(time: 25, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.BRANCH_NAME ?: 'N/A'} | APP_PORT=${env.APP_PORT}"
                script {
                    if (isUnix()) {
                        sh 'java -version || true; ls -la mvnw || true'
                    } else {
                        bat 'java -version & dir mvnw.cmd'
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    def mvnCmd = isUnix() ? './mvnw' : 'mvnw.cmd'
                    if (!fileExists(isUnix() ? 'mvnw' : 'mvnw.cmd')) {
                        error "Maven Wrapper not found. Add mvnw/mvnw.cmd to the repo (see instructions in job log)."
                    }
                    if (isUnix()) {
                        sh "${mvnCmd} ${MAVEN_ARGS} -DskipTests clean package"
                    } else {
                        bat "${mvnCmd} %MAVEN_ARGS% -DskipTests clean package"
                    }
                }
            }
        }

        stage('Unit Test') {
            steps {
                script {
                    def mvnCmd = isUnix() ? './mvnw' : 'mvnw.cmd'
                    if (isUnix()) {
                        sh "${mvnCmd} ${MAVEN_ARGS} test"
                    } else {
                        bat "${mvnCmd} %MAVEN_ARGS% test"
                    }
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Smoke Test (run on 9090)') {
            steps {
                script {
                    if (isUnix()) {
                        def artifact = sh(script: "ls -1 target/*.jar target/*.war 2>/dev/null | head -n 1", returnStdout: true).trim()
                        if (!artifact) { error "No artifact in target/. Did the Build stage succeed?" }
                        echo "Starting ${artifact} on port ${env.APP_PORT}"
                        sh """
                            nohup java -jar "${artifact}" --server.port=${env.APP_PORT} > app.log 2>&1 &
                            echo \$! > app.pid
                        """
                        sleep 12
                        sh """
                            for i in {1..15}; do
                              if curl -sSf "http://localhost:${env.APP_PORT}/" > /dev/null; then
                                echo "App is reachable on http://localhost:${env.APP_PORT}/"
                                exit 0
                              fi
                              sleep 2
                            done
                            echo "App did not become ready on port ${env.APP_PORT}"
                            exit 1
                        """
                    } else {
                        bat """
                            setlocal enabledelayedexpansion
                            set "ARTIFACT="
                            for %%i in (target\\*.jar) do if not defined ARTIFACT set "ARTIFACT=%%i"
                            if not defined ARTIFACT for %%i in (target\\*.war) do if not defined ARTIFACT set "ARTIFACT=%%i"
                            if not defined ARTIFACT (
                              echo No artifact found in target\\
                              exit /b 1
                            )
                            echo Starting !ARTIFACT! on port ${env.APP_PORT}
                            start "" cmd /c "java -jar \"!ARTIFACT!\" --server.port=${env.APP_PORT} 1> app.log 2>&1"
                            endlocal
                        """
                        sleep 15
                        bat """
                            for /l %%t in (1,1,20) do (
                                curl -s http://localhost:${env.APP_PORT}/ >nul 2>&1 && exit /b 0
                                ping -n 2 127.0.0.1 >nul
                            )
                            echo App did not become ready on port ${env.APP_PORT}
                            exit /b 1
                        """
                    }
                }
            }
            post {
                always {
                    script {
                        if (isUnix()) {
                            sh '''
                                if [ -f app.pid ]; then
                                  kill $(cat app.pid) 2>/dev/null || true
                                  rm -f app.pid
                                else
                                  pids=$(lsof -ti tcp:'"'"$APP_PORT"'"') || true
                                  [ -n "$pids" ] && kill $pids || true
                                fi
                            '''
                        } else {
                            bat """
                                for /f "tokens=5" %%p in ('netstat -ano ^| findstr ":%APP_PORT% " ^| findstr LISTENING') do taskkill /PID %%p /F >nul 2>&1
                            """
                        }
                    }
                    archiveArtifacts artifacts: 'target/*.jar, target/*.war, app.log', allowEmptyArchive: true, fingerprint: true
                }
            }
        }
    }

    post {
        success { echo "✅ Build, Unit Tests і Smoke Test пройшли успішно на порту ${env.APP_PORT}" }
        failure { echo "❌ Помилка пайплайна. Перевірте Console Output та app.log (архівовано)" }
        always  { cleanWs() }
    }
}
