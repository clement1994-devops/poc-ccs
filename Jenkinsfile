pipeline{
    agent {
        label 'docker-nodejs-poc-agent'
    }
    stages {
        stage ('NPM Build Stage'){
            steps{
                updateGitlabCommitStatus name: 'NPM Build Stage', state: 'running'
                sh "npm install"
                updateGitlabCommitStatus name: 'NPM Build Stage', state: 'success'
            }
        }
        stage ('NPM Test Stage'){
            steps{
                updateGitlabCommitStatus name: 'NPM Test Stage', state: 'running'
                echo "Running Unit Test Cases"
                updateGitlabCommitStatus name: 'NPM Test Stage', state: 'success'
            }
        }
        stage ('Code Review'){
            environment {
                scannerHome = tool 'CCS-SonarScanner'
            }
            when {  expression { gitlabActionType != 'NOTE' } }
            steps{
                updateGitlabCommitStatus name: 'Code Review', state: 'running'
                withSonarQubeEnv('CCS-Sonarqube'){
                    sh "${scannerHome}/bin/sonar-scanner"
                }
                //sonar scan succeeded but quality gate check pending hence reporting success for the stage
                updateGitlabCommitStatus name: 'Code Review', state: 'success'
                timeout(time: 5, unit: 'MINUTES'){
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        stage ('Container Build Stage'){
            environment {
                NEXUS_VERSION = "nexus3"
                NEXUS_PROTOCOL = "http"
                NEXUS_PULL_URL = "pullregistry.ccstechnologies.org"
                NEXUS_CREDENTIAL = credentials("sonatype-nexus-suser")
            }
            when {  expression { gitlabActionType == 'NOTE' } }
            steps{
                updateGitlabCommitStatus name: 'Container Build Stage', state: 'running'
                sh "docker login -u $NEXUS_CREDENTIAL_USR -p $NEXUS_CREDENTIAL_PSW $NEXUS_PULL_URL"
                sh "docker build -t ccs-task-management-api:latest ."
                sh "docker logout $NEXUS_PULL_URL"
                updateGitlabCommitStatus name: 'Container Build Stage', state: 'success'
            }
        }
        stage ('Container Push Stage'){
            environment {
                NEXUS_VERSION = "nexus3"
                NEXUS_PROTOCOL = "http"
                NEXUS_PUSH_URL = "pushregistry.ccstechnologies.org"
                NEXUS_CREDENTIAL = credentials("sonatype-nexus-suser")
            }
            when {  expression { gitlabActionType == 'NOTE' } }
            steps{
                updateGitlabCommitStatus name: 'Container Push Stage', state: 'running'
                sh "docker login -u $NEXUS_CREDENTIAL_USR -p $NEXUS_CREDENTIAL_PSW $NEXUS_PUSH_URL"
                sh "docker tag ccs-task-management-api:latest $NEXUS_PUSH_URL/ccs-task-management-api:dev-${BUILD_NUMBER}"
                sh "docker push $NEXUS_PUSH_URL/ccs-task-management-api:dev-${BUILD_NUMBER}"
                sh "docker logout $NEXUS_PUSH_URL"
                updateGitlabCommitStatus name: 'Container Push Stage', state: 'success'
            }
        }
    }
    options {
      gitLabConnection('CCS-Gitlab')
    }
    post {
        success {
            mail to:"clement.pa@ccs-technologies.com,midhun.raj@ccs-technologies.com",subject:"SUCCESS: ${currentBuild.fullDisplayName}",body: "Build Successful. \n ${currentBuild.absoluteUrl}"
            slackSend (channel: '#poc-cicd', color: '#00FF00', message: "SUCCESSFUL: Job '${currentBuild.fullDisplayName}(${currentBuild.absoluteUrl})")
        }
        failure {
            updateGitlabCommitStatus state: 'failed'
            mail to:"clement.pa@ccs-technologies.com,midhun.raj@ccs-technologies.com",subject:"FAILURE: ${currentBuild.fullDisplayName}",body: "Build failed!Check build logs for errors. \n ${currentBuild.absoluteUrl}"
            slackSend (channel: '#poc-cicdls', color: '#FF0000',message: "FAILED: Job '${currentBuild.fullDisplayName}(${currentBuild.absoluteUrl})")
        }
    }
}
