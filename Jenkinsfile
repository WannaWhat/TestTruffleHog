User
btained Jenkinsfile from 0c9d673c3336bbc8501b88fd445ba7016c34c689
org.codehaus.groovy.control.MultipleCompilationErrorsException: startup failed:
WorkflowScript: 33: Expected a step @ line 33, column 21.
                       if (result.contains("leaks found")) {
                       ^

1 error

	at org.codehaus.groovy.control.ErrorCollector.failIfErrors(ErrorCollector.java:309)
	at org.codehaus.groovy.control.CompilationUnit.applyToPrimaryClassNodes(CompilationUnit.java:1107)
	at org.codehaus.groovy.control.CompilationUnit.doPhaseOperation(CompilationUnit.java:624)
	at org.codehaus.groovy.control.CompilationUnit.processPhaseOperations(CompilationUnit.java:602)
	at org.codehaus.groovy.control.CompilationUnit.compile(CompilationUnit.java:579)
	at groovy.lang.GroovyClassLoader.doParseClass(GroovyClassLoader.java:323)
	at groovy.lang.GroovyClassLoader.parseClass(GroovyClassLoader.java:293)
	at org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.GroovySandbox$Scope.parse(GroovySandbox.java:163)
	at org.jenkinsci.plugins.workflow.cps.CpsGroovyShell.doParse(CpsGroovyShell.java:190)
	at org.jenkinsci.plugins.workflow.cps.CpsGroovyShell.reparse(CpsGroovyShell.java:175)
	at org.jenkinsci.plugins.workflow.cps.CpsFlowExecution.parseScript(CpsFlowExecution.java:635)
	at org.jenkinsci.plugins.workflow.cps.CpsFlowExecution.start(CpsFlowExecution.java:581)
	at org.jenkinsci.plugins.workflow.job.WorkflowRun.run(WorkflowRun.java:335)
	at hudson.model.ResourceController.execute(ResourceController.java:101)
	at hudson.model.Executor.run(Executor.java:442)
Finished: FAILURE

REST API


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
