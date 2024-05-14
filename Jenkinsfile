import groovy.json.JsonOutput

def sendTelegramNotification(String stage, String message) {
    def scmVars = checkout scm
    withCredentials([string(credentialsId: 'BotToken', variable: 'TELEGRAM_TOKEN'),
                     string(credentialsId: 'TelegramChatID', variable: 'TELEGRAM_CHAT_ID')]) {
        def jsonPayload = JsonOutput.toJson([
            chat_id: "${TELEGRAM_CHAT_ID}",
            text: "Job - '$JOB_NAME' ($BUILD_NUMBER) at stage '${stage}'\nCommit hash ${scmVars.GIT_COMMIT}\n\nMessage: ${message}"
        ])
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
        REGISTRY_PROTOCOL = 'http://'
    }
    parameters {
        string(name: 'IMAGE_NAME', defaultValue: 'testtrufflehog', description: 'Name of the Docker image')
        string(name: 'REGISTRY_URL', defaultValue: 'localhost:9001', description: 'Docker registry URL')
        booleanParam(name: 'CLEAN_BUILD', defaultValue: false, description: 'Check here to perform new build')
    }

    stages {
//     Scan code on secrets with GitLeaks
        stage('Run GitLeaks scan code on secrets') {
            when {
                expression {
                    params.CLEAN_BUILD
                }            }
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

//     Login into Docker registry
        stage("Login into docker registry") {
            steps {
                script {
                    docker.withRegistry("${env.REGISTRY_PROTOCOL}${params.REGISTRY_URL}", "${env.DOCKER_CREDENTIALS_ID}") {
                        echo 'Successfully logged in to Docker Registry'
                    }
                }
            }
        }

//     Build tmp_develop docker image to check build status
        stage("Build tmp_develop docker image") {
            when {
                expression {
                    params.CLEAN_BUILD
                }            }
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

//     Re-tag old docker develop image to lastwork_develop image
        stage("Re-tag develop docker image to lastwork_develop image") {
            when {
                expression {
                    params.CLEAN_BUILD
                }            }
            steps {
                script {
                    try {
                        sh "docker pull ${params.REGISTRY_URL}/${params.IMAGE_NAME}:develop"
                    } catch (Exception e) {
                        sendTelegramNotification("Retag develop -> lastwork_develop", "No develop image - Skip")
                        return
                    }
                    sh "docker tag ${params.REGISTRY_URL}/${params.IMAGE_NAME}:develop ${params.REGISTRY_URL}/${params.IMAGE_NAME}:lastwork_develop"
                }
            }
        }

//     Re-tag tmp_develop docker image to develop image for future deploy
        stage("Re-tag tmp_develop docker image to develop image") {
            when {
                expression {
                    params.CLEAN_BUILD
                }
            }
            steps {
                script {
                    sh "docker tag ${params.IMAGE_NAME}:tmp_develop ${params.REGISTRY_URL}/${params.IMAGE_NAME}:develop"
                    sh "docker push ${params.REGISTRY_URL}/${params.IMAGE_NAME}:develop"
                }
            }
        }

//     Re-tag lastwork_develop to develop image to renew old build
        stage("Re-tag lastwork_develop to develop docker image") {
            when {
                not {
                    expression {
                        params.CLEAN_BUILD
                    }
                }
            }
            steps {
                script {
                    try {
                        sh "docker pull ${params.REGISTRY_URL}/${params.IMAGE_NAME}:lastwork_develop"
                    } catch (Exception e) {
                        sendTelegramNotification("Re-tag lstw_develop to develop", "No lastwork_develop image - found")
                        return
                    }
                    sh "docker tag ${params.REGISTRY_URL}/${params.IMAGE_NAME}:lastwork_develop ${params.REGISTRY_URL}/${params.IMAGE_NAME}:lastwork_develop"
                }
            }
        }
    }
    post {
        always {
            script {
                def status = currentBuild.result ?: 'SUCCESS'
                echo "${status}"
                if ("${status}" != 'SUCCESS') {
                    sendTelegramNotification("Post script", "Build failed with status: ${status}")
                } else {
                    sendTelegramNotification("Post script", "Build completed with status: ${status}")
                }

            }
        }
    }
}
