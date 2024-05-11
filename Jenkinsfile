pipeline {
    agent any

    stages {
        stage('Show echo') {
            steps {
                script {
                    sh 'whoami'
                }
            }
        }
        stage('Run GitLeaks') {
            steps {
                script {
                    def result = sh(script: 'docker run --rm -v $(pwd):/code zricethezav/gitleaks detect --source=/code --verbose --redact', returnStdout: true).trim()
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
                if (status != 'SUCCESS') {
                    sendTelegramNotification("Post script", "Build failed with status: ${status}")
                }
            }
        }
    }
}
