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
    // githubPush() still fires on every push (branches AND tags).
    // The branch spec 'refs/tags/*' in the checkout below is what actually
    // filters this out: a plain branch push (e.g. a PR merge to staging)
    // will NOT match that spec, so Jenkins won't check anything out or build.
    // Only an actual `git push origin <tag>` matches and triggers a real build.
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
                    echo ":small_blue_diamond: Checking out pushed tag"
                    def scmVars = checkout([$class: 'GitSCM',
                        branches: [[name: 'refs/tags/*']],
                        userRemoteConfigs: [[
                            url: env.GIT_REPO,
                            credentialsId: env.GIT_CREDENTIALS_ID,
                            // Explicit refspec ensures tags are actually fetched
                            // into refs/remotes/origin/tags/* so the branch spec above can match them.
                            refspec: '+refs/tags/*:refs/remotes/origin/tags/*'
                        ]]
                    ])
                    // scmVars.GIT_BRANCH looks like "origin/tags/v1.2.11-rc1" for a tag checkout
                    env.ACTUAL_REF = scmVars.GIT_BRANCH
                }
            }
        }
        stage('Environment Info') {
            steps {
                echo """
                Environment Info
                ----------------------
                Checked out ref: ${env.ACTUAL_REF}
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
                        // Because we only ever check out refs/tags/*, this ref
                        // is guaranteed to be a tag if the pipeline got this far.
                        if (!env.ACTUAL_REF || !env.ACTUAL_REF.contains('tags/')) {
                            error("No tag was checked out. This pipeline only builds on tag pushes.")
                        }
                        imageTag = env.ACTUAL_REF.tokenize('/').last()
                    }
                    env.IMAGE_TAG = imageTag
                    echo ":rocket: FINAL Docker Tag: ${env.IMAGE_TAG}"
                }
            }
        }
    }
}