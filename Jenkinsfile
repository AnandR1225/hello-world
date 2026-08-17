pipeline {
    agent any
    options {
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }
    environment {
        GIT_REPO               = "https://github.com/AnandR1225/hello-world.git"
        GIT_CREDENTIALS_ID     = "github1"
        // DOCKER_CREDENTIALS_ID  = "docker-report"
        // SONARQUBE_ENV          = "sonar-server"
        // NAMESPACE              = "reports"
        // DEPLOY_ENV             = "production"
        // IMAGE_NAME             = "prophazedocker/i-report"
        // KUBERNETES_CREDENTIALS_ID = "k3s-report-staging"
        // DEPLOYMENT_FILE        = "prod-reports.yaml"
        // DEPLOYMENT_NAME        = "prod-reports-api"
    }
    parameters {
        booleanParam(name: 'ROLLBACK', defaultValue: false, description: 'Rollback to TARGET_VERSION instead of deploy')
        string(name: 'TARGET_VERSION', defaultValue: '', description: 'Target Docker tag for rollback (if enabled)')
    }
    triggers {
        githubPush()
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout Code') {
            steps {
                script {
                    echo ":small_blue_diamond: Checking out branch: master"
                    checkout([$class: 'GitSCM',
                        branches: [[name: "*/staging"]],
                        userRemoteConfigs: [[
                            url: env.GIT_REPO,
                            credentialsId: env.GIT_CREDENTIALS_ID
                        ]]
                    ])
                    env.ACTUAL_BRANCH = "staging"
                }
            }
        }
        stage('Environment Info') {
            steps {
                echo """
                Environment Info
                ----------------------
                Branch: ${env.ACTUAL_BRANCH}
                Deploy: ${env.DEPLOY_ENV}
                Repo:   ${env.IMAGE_NAME}
                Namespace: ${env.NAMESPACE}
                Deployment File: ${env.DEPLOYMENT_FILE}
                """
            }
        }

        stage('Generate Docker Tag') {
            steps {
                script {
                    def imageTag = ""
                    if (params.ROLLBACK) {
                        if (!params.TARGET_VERSION?.trim()) {
                            error("Rollback requested but no TARGET_VERSION provided.")
                        }
                        imageTag = params.TARGET_VERSION.trim()
                    } else {
                        def tagName = sh(
                            script: "git describe --tags --exact-match HEAD 2>/dev/null || true",
                            returnStdout: true
                        ).trim()
                        if (!tagName) {
                          error("Tag not found. Stopping build.")
                        }
                        imageTag = tagName
                    }
                    env.IMAGE_TAG = imageTag
                    echo ":rocket: FINAL Docker Tag: ${env.IMAGE_TAG}"
                }
            }
        }
    }
}


// testing jenkinsfile