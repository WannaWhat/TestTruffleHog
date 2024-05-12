def sendTelegramNotification(String stage, String message) {
    def scmVars = checkout scm
    withCredentials([string(credentialsId: 'BotToken', variable: 'TELEGRAM_TOKEN'),
                     string(credentialsId: 'TelegramChatID', variable: 'TELEGRAM_CHAT_ID')]) {
        def jsonPayload = """
        {
            "chat_id": "${TELEGRAM_CHAT_ID}",
            "text": "Job - '$JOB_NAME' ($BUILD_NUMBER) at stage '${stage}'\nCommit hash ${scmVars.GIT_COMMIT}\n\nMessage: ${message}"
        }
        """
        httpRequest url: "https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: jsonPayload
    }
}


pipeline {
    agent any
    stages {
        stage('Run GitLeaks scan code on secrets') {
            steps {
                script {
                    def status = sh(script: "gitleaks detect --source=. --verbose --no-git --redact > output.txt 2>&1", returnStatus: true)
                    def output = readFile('output.txt').trim()
                    echo "${output}"
                    if (output.contains("no leaks found")) {
                        return
                    }
                    sendTelegramNotification("Run GitLeaks", "Leaks - found")
                    error("Leaks found")
                }
            }
        }
        stage("Build tmp_develop docker image") {
            steps {
                script {
                    if (fileExists('Dockerfile')) {
                        docker.build("my-image:tmp_develop")
                    } else {
                        error("Dockerfile not found")
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                def status = currentBuild.result ?: 'SUCCESS'
                if ("${status}" != 'SUCCESS') {
                    sendTelegramNotification("Post script", "Build failed with status: ${status}")
                }
                sendTelegramNotification("Post script", "Build completed with status: ${status}")

            }
        }
    }
}
