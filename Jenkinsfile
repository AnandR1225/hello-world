pipeline {
    agent any

    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        DOCKER_REPO           = "anand/hello-world"
        DOCKER_CREDENTIALS_ID = "docker-creds"
        KUBE_CRED             = "kube-cred"
    }

    // No SCM-level `triggers { githubPush() }` needed when using a
    // Multibranch Pipeline with GitHub Branch Source + "Discover tags" —
    // the branch/tag indexing itself acts as the trigger. Adding
    // githubPush() here is harmless (it just re-registers the hook) but
    // is not what causes tag runs to fire; the branch source behavior is.

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Gate: Tag builds only') {
            steps {
                script {
                    if (!env.TAG_NAME?.trim()) {
                        currentBuild.result = 'NOT_BUILT'
                        error("Not a tag build (BRANCH_NAME=${env.BRANCH_NAME}) — aborting before any build/push/deploy stage.")
                    }

                    if (env.TAG_NAME ==~ /^v?\d+\.\d+\.\d+-rc\d+$/) {
                        env.DEPLOY_ENV = "staging"
                        env.NAMESPACE  = "app-staging"
                    } else if (env.TAG_NAME ==~ /^v?\d+\.\d+\.\d+$/) {
                        env.DEPLOY_ENV = "prod"
                        env.NAMESPACE  = "app-prod"
                    } else {
                        error("Tag '${env.TAG_NAME}' doesn't match vX.Y.Z or vX.Y.Z-rcN — aborting.")
                    }

                    env.IMAGE_TAG = env.TAG_NAME
                    echo "Tag=${env.TAG_NAME} -> env=${env.DEPLOY_ENV} ns=${env.NAMESPACE} image_tag=${env.IMAGE_TAG}"
                }
            }
        }

        stage('Build Image') {
            steps {
                sh "docker build -t ${DOCKER_REPO}:${IMAGE_TAG} ."
            }
        }
    }

    post {
        success {
            echo "Deployed ${env.TAG_NAME ?: 'n/a'} to ${env.DEPLOY_ENV ?: 'n/a'}"
        }
        not_built {
            echo "Skipped: non-tag ref (${env.BRANCH_NAME}), no build/deploy performed."
        }
        failure {
            echo "Failed for ${env.TAG_NAME ?: env.BRANCH_NAME}"
        }
    }
}

/// jenkins tags might be working