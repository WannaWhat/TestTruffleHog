import tools.teleGramFunctions


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
                    def status = sh(script: "gitleaks detect --source=. --verbose --no-git --redact > output.txt 2>&1", returnStatus: true)
                    def output = readFile('output.txt').trim()
                    if (output.contains("no leaks found")) {
                        return
                    }
                    teleGramFunctions.sendTelegramNotification("Run GitLeaks", "Leaks - found")
//                     sendTelegramNotification("Run GitLeaks", "Leaks - found")
                    error("Leaks found")
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
