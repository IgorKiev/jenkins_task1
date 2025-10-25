pipeline {
    agent any

    // Якщо налаштовані Global Tools у Jenkins, розкоментуйте і підставте свої назви:
    // tools {
    //     jdk   'jdk8'
    //     maven 'maven-3.9.9'
    // }

    environment {
        APP_PORT   = '9090'
        MAVEN_ARGS = '-B -U'   // non-interactive, update snapshots
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
                    if (isUnix()) { sh 'java -version || true; which mvn || true; ls -la mvnw || true' }
                    else          { bat 'java -version & where mvn & dir mvnw.cmd' }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    // Визначаємо команду Maven: wrapper, якщо є; інакше системний mvn
                    def mvnCmd = isUnix()
                        ? (fileExists('mvnw')     ? './mvnw'    : 'mvn')
                        : (fileExists('mvnw.cmd') ? 'mvnw.cmd'  : 'mvn')

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
                    def mvnCmd = isUnix()
                        ? (fileExists('mvnw')     ? './mvnw'    : 'mvn')
                        : (fileExists('mvnw.cmd') ? 'mvnw.cmd'  : 'mvn')

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
                        // ---------- Linux/macOS ----------
                        // Знаходимо артефакт
                        def artifact = sh(script: "ls -1 target/*.jar target/*.war 2>/dev/null | head -n 1", returnStdout: true).trim()
                        if (!artifact) { error "No artifact in target/. Did the Build stage succeed?" }

                        echo "Starting ${artifact} on port ${env.APP_PORT}"
                        // Запуск у фоні; уникаємо export і зайвих $ у Groovy
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
                        // ---------- Windows ----------
                        // Пошук артефакта (спершу JAR, потім WAR)
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
                        // Перевірка доступності (curl у Win10+ є)
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
                                  # fallback: kill by port
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
