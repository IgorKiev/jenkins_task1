pipeline {
    agent any

    // Порт застосунку
    environment {
        APP_PORT   = '9090'
        MAVEN_ARGS = '-B -U'     // non-interactive, update snapshots
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
                        sh 'java -version || true; mvn -v || true'
                    } else {
                        bat 'java -version & mvn -v'
                    }
                }
            }
        }

        stage('Build') {
            steps {
                // Use the maven package phase to build the project
                script {
                    if (isUnix()) {
                        sh "mvn ${MAVEN_ARGS} -DskipTests clean package"
                    } else {
                        bat "mvn %MAVEN_ARGS% -DskipTests clean package"
                    }
                }
            }
        }

        stage('Unit Test') {
            steps {
                // Use the maven test phase to run unit tests
                script {
                    if (isUnix()) {
                        sh "mvn ${MAVEN_ARGS} test"
                    } else {
                        bat "mvn %MAVEN_ARGS% test"
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
                    // ---- Запуск на Linux ----
                    if (isUnix()) {
                        // Спробуємо JAR, якщо нема — WAR
                        sh """
                            set -e
                            export APP_PORT=${APP_PORT}
                            ARTIFACT=""
                            if ls target/*.jar >/dev/null 2>&1; then ARTIFACT=$(ls -1 target/*.jar | head -n 1); fi
                            if [ -z "$ARTIFACT" ] && ls target/*.war >/dev/null 2>&1; then ARTIFACT=$(ls -1 target/*.war | head -n 1); fi
                            if [ -z "$ARTIFACT" ]; then
                              echo "No artifact found in target/. Run Build stage first."
                              exit 1
                            fi
                            echo "Starting $ARTIFACT on port ${APP_PORT}"
                            nohup java -jar "$ARTIFACT" --server.port=${APP_PORT} > app.log 2>&1 &
                            echo \$! > app.pid
                        """
                        sleep 12
                        sh """
                            for i in {1..15}; do
                              if curl -sSf "http://localhost:${APP_PORT}/" > /dev/null; then
                                echo "App is reachable on http://localhost:${APP_PORT}/"
                                exit 0
                              fi
                              sleep 2
                            done
                            echo "App did not become ready on port ${APP_PORT}"
                            exit 1
                        """

                    // ---- Запуск на Windows ----
                    } else {
                        // Запускаємо артефакт (спочатку JAR, потім WAR), лог у app.log
                        bat """
                            set APP_PORT=${APP_PORT}
                            set ARTIFACT=
                            for %%i in (target\\*.jar) do ( if not defined ARTIFACT set ARTIFACT=%%i )
                            if not defined ARTIFACT (
                                for %%i in (target\\*.war) do ( if not defined ARTIFACT set ARTIFACT=%%i )
                            )
                            if not defined ARTIFACT (
                                echo No artifact found in target\\. Run Build stage first.
                                exit /b 1
                            )
                            echo Starting %ARTIFACT% on port %APP_PORT%
                            start "" cmd /c "java -jar \"%ARTIFACT%\" --server.port=%APP_PORT% 1> app.log 2>&1"
                        """
                        // Даємо час піднятися й перевіряємо доступність
                        sleep 15
                        // Перевірка через curl (Win10+). Якщо нема curl, можна замінити на PowerShell Invoke-WebRequest.
                        bat """
                            set APP_PORT=${APP_PORT}
                            for /l %%t in (1,1,10) do (
                                curl -s http://localhost:%APP_PORT%/ >nul 2>&1 && exit /b 0
                                ping -n 2 127.0.0.1 >nul
                            )
                            echo App did not become ready on port %APP_PORT%
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
                                  # fallback: kill by port
                                  pids=$(lsof -ti tcp:'"'$APP_PORT'"') || true
                                  [ -n "$pids" ] && kill $pids || true
                                fi
                            '''
                        } else {
                            // Вбиваємо процес, що слухає порт
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
        failure { echo "❌ Помилка пайплайна. Перевірте Console Output і app.log (архівовано)" }
        always  { cleanWs() }
    }
}
