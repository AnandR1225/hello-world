// This Jenkinsfile assumes the Multibranch Pipeline config is set to:
//   Behaviors: Discover tags (only) -- 'Discover branches' removed
//   Filter by name (with wildcards): Include = v*
//   Scan by webhook, pointed at /multibranch-webhook-trigger/invoke?token=...
//
// With that in place, Multibranch creates one job per matching tag and
// checks it out automatically before this Jenkinsfile runs. No manual
// checkout stage is needed -- env.TAG_NAME is set by Multibranch itself
// when the currently-building item is a discovered tag.

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
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        // No custom checkout: Multibranch already checked out this exact
        // tag before the pipeline started, because this job only exists
        // for tags matching the 'v*' filter.
        stage('Environment Info') {
            steps {
                echo """
                Environment Info
                ----------------------
                Tag:    ${env.TAG_NAME}
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
                    } else if (env.TAG_NAME?.trim()) {
                        // Set directly by Multibranch when this item is a discovered tag.
                        imageTag = env.TAG_NAME.trim()
                    } else {
                        // Fallback for safety, e.g. manual "Build Now" replay.
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