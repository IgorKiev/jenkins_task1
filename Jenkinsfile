pipeline {
    // Use any agent
    agent any

    // (Опційно) менеджовані інструменти Jenkins — змініть назви під ваші
    tools {
        jdk   'JDK8'     // ваш інстальований у Jenkins JDK 8
        maven 'Maven3'   // ваш інстальований у Jenkins Maven
    }

    // Set the environment variable APP_PORT=9090
    environment {
        APP_PORT   = '9090'
        MAVEN_ARGS = '-B -U'  // non-interactive, оновлення snapshots
    }

    options {
        ansiColor('xterm')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
        timeout(time: 25, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.BRANCH_NAME ?: 'N/A'} | APP_PORT=${env.APP_PORT}"
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
                    // Публікуємо результати Surefire, навіть якщо тестів немає
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Smoke Test (run on 9090)') {
            when {
                expression {
                    // Виконаємо лише якщо після білду є артефакт
                    def cnt = isUnix()
                        ? sh(script: "ls -1 target/*.war target/*.jar 2>/dev/null | wc -l", returnStdout: true).trim()
                        : bat(script: "for %i in (target\\*.war target\\*.jar) do @set /a c+=1 & @echo %i>nul & @echo !c!", returnStdout: true)
                    return (cnt?.isNumber() && cnt.toInteger() > 0) || (cnt && cnt.trim() != "0")
                }
            }
            steps {
                script {
                    // Знаходимо перший артефакт
                    def artifact = isUnix()
                        ? sh(script: 'ls -1 target/*.war target/*.jar 2>/dev/null | head -n 1', returnStdout: true).trim()
                        : bat(script: 'for %%i in (target\\*.war target\\*.jar) do (echo %%i & goto :found)\n:found', returnStdout: true).trim().split('\\r?\\n')[-1]

                    if (!artifact) {
                        error "Артефакт не знайдено в target/. Перевірте стадію Build."
                    }

                    echo "Starting ${artifact} on port ${env.APP_PORT}"

                    if (isUnix()) {
                        // Запускаємо Spring Boot WAR/JAR з потрібним портом
                        sh """
                            set -e
                            export APP_PORT=${APP_PORT}
                            nohup java -jar "${artifact}" --server.port=${APP_PORT} > app.log 2>&1 &
                            echo \$! > app.pid
                        """
                        // Чекаємо на старт і перевіряємо доступність
                        sleep 10
                        sh '''
                            set -e
                            for i in {1..15}; do
                              if curl -sSf "http://localhost:'"${APP_PORT}"'/" > /dev/null; then
                                echo "App is reachable on http://localhost:'"${APP_PORT}"'/"
                                exit 0
                              fi
                              sleep 2
                            done
                            echo "App did not become ready on port '${APP_PORT}'"
                            exit 1
                        '''
                    } else {
                        // Windows (PowerShell/CMD)
                        bat """
                            set APP_PORT=${APP_PORT}
                            start /b cmd /c java -jar "${artifact}" --server.port=%APP_PORT% 1> app.log 2>&1
                            @for /f "tokens=2" %%p in ('wmic process where "CommandLine like '%%java%%--server.port=%APP_PORT%%%'" get ProcessId ^| findstr /r "^[0-9]"') do @echo %%p> app.pid
                        """
                        sleep 15
                        // простий curl з PowerShell (якщо curl недоступний, у Win10+ він є за замовч.)
                        bat """
                            for /l %%i in (1,1,10) do (
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
                        // Зупиняємо процес, якщо він запущений
                        if (isUnix()) {
                            sh '''
                                if [ -f app.pid ]; then
                                  kill $(cat app.pid) 2>/dev/null || true
                                  rm -f app.pid
                                fi
                            '''
                        } else {
                            bat """
                                if exist app.pid (
                                  for /f %%p in (app.pid) do taskkill /PID %%p /F >nul 2>&1
                                  del /f /q app.pid
                                )
                            """
                        }
                    }
                    // Архівуємо логи та артефакти
                    archiveArtifacts artifacts: 'target/*.jar, target/*.war, app.log', allowEmptyArchive: true, fingerprint: true
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build, Unit Tests та Smoke Test пройшли успішно. Порт: ${env.APP_PORT}"
        }
        failure {
            echo "❌ Пайплайн впав. Перевірте консолі лог і app.log (архівовано)."
        }
        always {
            cleanWs()
        }
    }
}
