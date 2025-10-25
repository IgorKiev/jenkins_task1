pipeline {
    // Use any agent
    agent any

    // Set the environment variable APP_PORT=9090
    environment {
        APP_PORT   = '9090'
        MAVEN_ARGS = '-B -U'  // non-interactive, оновлення snapshots
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
                // Додатково покажемо версії інструментів
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
                    // Публікуємо результати Surefire
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Smoke Test (run on 9090)') {
            when {
                expression {
                    // Виконуємо лише якщо існує артефакт у target/
                    def cmd = isUnix()
                        ? "bash -lc 'ls -1 target/*.war target/*.jar 2>/dev/null | wc -l'"
                        : "cmd /c for %i in (target\\*.war target\\*.jar) do @set /a c+=1 & @echo %c%>count.txt & @type count.txt"
                    def out = isUnix()
                        ? sh(script: cmd, returnStdout: true).trim()
                        : bat(script: cmd, returnStdout: true).trim()
                    return out.isNumber() ? out.toInteger() > 0 : (out && out != '0')
                }
            }
            steps {
                script {
                    // Знаходимо артефакт
                    def artifact = isUnix()
                        ? sh(script: 'ls -1 target/*.war target/*.jar 2>/dev/null | head -n 1', returnStdout: true).trim()
                        : bat(script: '@for %%i in (target\\*.war target\\*.jar) do (echo %%i & goto :found)\n:found', returnStdout: true).trim().split('\\r?\\n')[-1]

                    if (!artifact) {
                        error "Артефакт не знайдено в target/. Перевірте стадію Build."
                    }

                    echo "Starting ${artifact} on port ${env.APP_PORT}"

                    if (isUnix()) {
                        // Запуск у бекграунді з логом
                        sh """
                            set -e
                            export APP_PORT=${APP_PORT}
                            nohup java -jar "${artifact}" --server.port=${
