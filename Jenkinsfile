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
    parameters {
        string(imageName: 'IMAGE_NAME', defaultValue: 'test_truffle_hog', description: 'Name of the Docker image')
    }
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
//                     sendTelegramNotification("Run GitLeaks", "Leaks - found")
                    return "Leaks - found"
                    error("Leaks found")
                }
            }
        }
        stage("Build tmp_develop docker image") {
            steps {
                script {
                    if (fileExists('Dockerfile')) {
                        docker.build("${params.CONTAINER_NAME}:tmp_develop")
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
                    def error = currentBuild.rawBuild.getActions(hudson.model.CauseAction.class)[0].getCauses()[0].getShortDescription()
                    sendTelegramNotification("Post script", "Build failed with error: ${error}")
                } else {
                    sendTelegramNotification("Post script", "Build completed with status: ${status}")
                }

            }
        }
    }
}
