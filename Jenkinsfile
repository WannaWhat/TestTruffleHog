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
    environment {
        DOCKER_CREDENTIALS_ID = 'DockerRegistryCred'
    }
    parameters {
        string(name: 'IMAGE_NAME', defaultValue: 'test_truffle_hog', description: 'Name of the Docker image')
        string(name: 'REGISTRY_URL', defaultValue: 'localhost:9001', description: 'Docker registry URL')
        string(name: 'REGISTRY_PROTOCOL', defaultValue: 'http://', description: 'Docker registry web protocol')
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
                    sendTelegramNotification("Run GitLeaks", "Leaks - found")
                    error("Leaks found")
                }
            }
        }
        stage("Login into docker registry") {
            steps {
                script {
                    docker.withRegistry("${params.REGISTRY_PROTOCOL}${params.REGISTRY_URL}", "${env.DOCKER_CREDENTIALS_ID}") {
                        echo 'Successfully logged in to Docker Registry'
                    }
                }
            }
        }
        stage("Build tmp_develop docker image") {
            steps {
                script {
                    if (fileExists('Dockerfile')) {
                        docker.build("${params.IMAGE_NAME}:tmp_develop")
                    } else {
                        error("Dockerfile not found")
                    }
                }
            }
        }
        stage("Retag develop docker image to lastwork_develop image") {
            steps {
                script {
                    try {
                        def originalImage = docker.image("${params.REGISTRY_URL}/${params.IMAGE_NAME}:develop")
                        originalImage.pull()
                    } catch (Exception e) {
                        sendTelegramNotification("Retag develop -> lastwork_develop", "No develop image - Skip")
                        return
                    }
                    def newImage = originalImage.tag("${params.IMAGE_NAME}:lastwork_develop")
                    newImage.push()
                }
            }
        }
        stage("Retag tmp_develop docker image to develop image") {
            steps {
                script {
                    def originalImage = docker.image("${params.IMAGE_NAME}:tmp_develop")
//                     originalImage.pull()
                    def newImage = originalImage.tag("${params.IMAGE_NAME}:develop")
                    newImage.push()
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
                } else {
                    sendTelegramNotification("Post script", "Build completed with status: ${status}")
                }

            }
        }
    }
}
