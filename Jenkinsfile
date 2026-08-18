pipeline {
    agent any

    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        DOCKER_REPO           = "prophazedocker/ml-api-discovery"
        DOCKER_CREDENTIALS_ID = "prophaze-docker"
        KUBE_CRED             = "ml-api"
        NAMESPACE             = "ml-api-discovery-prod"
        SOURCE_BRANCH         = "staging"
        AWS_REGION            = "ap-south-1"
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                withEnv(['GIT_LFS_SKIP_SMUDGE=1']) {
                    checkout scm
                }

                sh '''
                    set -e

                    echo "=========================================="
                    echo "Git Information"
                    echo "=========================================="

                    git log -1 --oneline
                    git rev-parse HEAD

                    echo ""
                    echo "Tags on current commit:"
                    git tag --points-at HEAD || true

                    echo "=========================================="
                '''
            }
        }

        stage('Validate Production Tag') {
            steps {
                script {

                    def tagName = sh(
                        script: '''
                            git tag --points-at HEAD | head -n 1
                        ''',
                        returnStdout: true
                    ).trim()

                    if (!tagName && env.TAG_NAME?.trim()) {
                        tagName = env.TAG_NAME.trim()
                    }

                    if (!tagName) {
                        error("No Git tag found. Production deployments require a version tag.")
                    }

                    if (!(tagName ==~ /^v\d+\.\d+\.\d+$/)) {
                        error(
                            "Invalid production tag '${tagName}'. " +
                            "Expected format: vX.Y.Z, for example v1.2.0"
                        )
                    }

                    env.RELEASE_TAG = tagName
                    env.IMAGE_TAG   = tagName
                    env.TAG_COMMIT  = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Validate Main Source') {
            steps {
                script {

                    sh '''
                        set -e

                        git fetch origin \
                            main:refs/remotes/origin/main
                    '''

                    def mainCommit = sh(
                        script: 'git rev-parse origin/main',
                        returnStdout: true
                    ).trim()

                    echo "Main HEAD: ${mainCommit}"
                    echo "Tagged commit: ${env.TAG_COMMIT}"

                    def result = sh(
                        script: """
                            git merge-base \
                                --is-ancestor \
                                ${env.TAG_COMMIT} \
                                origin/main
                        """,
                        returnStatus: true
                    )

                    if (result != 0) {
                        error(
                            "Production tag ${env.RELEASE_TAG} does not " +
                            "belong to the main branch history."
                        )
                    }

                    echo "Production source validation passed."
                }
            }
        }

        stage('Detect Changed Services') {
            steps {
                script {

                    sh '''
                        set -e
                        git fetch --tags --force
                    '''
                    def previousTag = sh(
                        script: """
                            git tag --sort=-creatordate \
                                --list 'v[0-9]*.[0-9]*.[0-9]*' \
                                | grep -Fxv '${env.RELEASE_TAG}' \
                                | grep -E '^v[0-9]+\\.[0-9]+\\.[0-9]+$' \
                                | head -n 1 || true
                        """,
                        returnStdout: true
                    ).trim()

                    def baseRef

                    if (previousTag) {

                        baseRef = previousTag

                        echo "Previous production tag: ${previousTag}"

                    } else {

                        baseRef = sh(
                            script: "git rev-list --max-parents=0 ${env.RELEASE_TAG}",
                            returnStdout: true
                        ).trim()

                        echo "No previous production tag found."
                        echo "Using initial commit: ${baseRef}"
                    }

                    def changedFiles = sh(
                        script: """
                            git diff \
                                --name-only \
                                ${baseRef} \
                                ${env.RELEASE_TAG}
                        """,
                        returnStdout: true
                    ).trim()

                    env.BUILD_API = changedFiles.contains(
                        "api_discovery/api_discovery_v1/"
                    ) ? "true" : "false"

                    env.BUILD_TOKENIZER = changedFiles.contains(
                        "api_discovery/tokenizer/"
                    ) ? "true" : "false"

                    if (
                        env.BUILD_API != "true" &&
                        env.BUILD_TOKENIZER != "true"
                    ) {
                        error(
                            "No API or Tokenizer changes detected. " +
                            "Nothing to deploy."
                        )
                    }

                    echo "BUILD_API=${env.BUILD_API}"
                    echo "BUILD_TOKENIZER=${env.BUILD_TOKENIZER}"
                }
            }
        }
    }
}