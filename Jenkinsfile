def sendTelegramNotification(String stage, String message) {
    def scmVars = checkout scm
    withCredentials([string(credentialsId: 'BotToken', variable: 'TELEGRAM_TOKEN'),
                     string(credentialsId: 'TelegramChatID', variable: 'TELEGRAM_CHAT_ID')]) {
        def jsonPayload = """
        {
            "chat_id": "${TELEGRAM_CHAT_ID}",
            "text": "Build - ${status}\nJob - '$JOB_NAME' ($BUILD_NUMBER) at stage '${stage}'\nCommit hash ${scmVars.GIT_COMMIT}\n\nMessage: ${message}"
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
        stage('Show echo') {
            steps {
                script {
                    sh 'whoami'
                    sh 'ls'
                }
            }
        }
        stage('Run GitLeaks') {
            steps {
                script {
                    def result = sh(script: 'docker run --rm -v $(pwd):/code zricethezav/gitleaks detect --source=/code --verbose --config /code/gitleaks.toml', returnStdout: true).trim()
                    if (result.contains("leaks found")) {
                        sendTelegramNotification("Run GitLeaks", "Leaks - found")
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                def status = currentBuild.result ?: 'SUCCESS'
                echo ${status}
                if (status != 'SUCCESS') {
                    sendTelegramNotification("Post script", "Build failed with status: ${status}")
                }
            }
        }
    }
}
