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

                    echo "Commit:"
                    git log -1 --oneline

                    echo ""
                    echo "Commit SHA:"
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
                        error(
                            "No Git tag found. " +
                            "Production deployments require a version tag."
                        )
                    }

                    /*
                     * Production tags must be:
                     *
                     * v1.0.0
                     * v1.2.0
                     * v10.25.100
                     *
                     * RC tags such as v1.2.0-rc1 are rejected.
                     */
                    if (!(tagName ==~ /^v\d+\.\d+\.\d+$/)) {
                        error(
                            "Invalid production tag '${tagName}'. " +
                            "Expected format vX.Y.Z, for example v1.2.0"
                        )
                    }

                    env.RELEASE_TAG = tagName
                    env.IMAGE_TAG = tagName

                    env.TAG_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    echo """
==========================================
PRODUCTION RELEASE
==========================================

Tag:
${env.RELEASE_TAG}

Commit:
${env.TAG_COMMIT}

Source Branch:
${env.SOURCE_BRANCH}

Environment:
production

Namespace:
${env.NAMESPACE}

==========================================
"""
                }
            }
        }

        stage('Validate Source Branch') {
            steps {
                script {

                    sh '''
                        set -e

                        git fetch origin \
                            staging:refs/remotes/origin/staging
                    '''

                    def sourceCommit = sh(
                        script: 'git rev-parse origin/staging',
                        returnStdout: true
                    ).trim()

                    echo "Staging HEAD: ${sourceCommit}"
                    echo "Tagged commit: ${env.TAG_COMMIT}"

                    /*
                     * Verify that the tagged commit exists
                     * in the staging branch history.
                     */
                    def result = sh(
                        script: """
                            git merge-base \
                                --is-ancestor \
                                ${env.TAG_COMMIT} \
                                origin/staging
                        """,
                        returnStatus: true
                    )

                    if (result != 0) {
                        error(
                            "Production tag ${env.RELEASE_TAG} does not " +
                            "belong to the staging branch history. " +
                            "Production deployment stopped."
                        )
                    }

                    echo """
==========================================
SOURCE VALIDATION PASSED
==========================================

Tag:
${env.RELEASE_TAG}

Source Branch:
${env.SOURCE_BRANCH}

Tagged Commit:
${env.TAG_COMMIT}

Staging HEAD:
${sourceCommit}

==========================================
"""
                }
            }
        }

//         stage('Detect Changed Services') {
//             steps {
//                 script {

//                     sh '''
//                         set -e
//                         git fetch --tags --force
//                     '''

//                     /*
//                      * Find previous production tag.
//                      *
//                      * We use shell filtering here instead of putting
//                      * the regex inside a Groovy double-quoted string.
//                      */
//                     def previousTag = sh(
//                         script: """
//                             git tag --sort=-creatordate --list 'v*' |
//                             grep -v -F -- '${env.RELEASE_TAG}' |
//                             grep -E '^v[0-9]+\\\\.[0-9]+\\\\.[0-9]+$' |
//                             head -n 1 || true
//                         """,
//                         returnStdout: true
//                     ).trim()

//                     def baseRef

//                     if (previousTag) {

//                         baseRef = previousTag

//                         echo "Previous production tag: ${previousTag}"

//                     } else {

//                         baseRef = sh(
//                             script: "git rev-list --max-parents=0 ${env.RELEASE_TAG}",
//                             returnStdout: true
//                         ).trim()

//                         echo "No previous production tag found."
//                         echo "Using initial commit: ${baseRef}"
//                     }

//                     def changedFiles = sh(
//                         script: """
//                             git diff \
//                                 --name-only \
//                                 ${baseRef} \
//                                 ${env.RELEASE_TAG}
//                         """,
//                         returnStdout: true
//                     ).trim()

//                     echo """
// ==========================================
// CHANGED FILES
// ==========================================

// ${changedFiles ?: 'No changed files detected'}

// ==========================================
// """

//                     env.BUILD_API =
//                         changedFiles.contains(
//                             "api_discovery/api_discovery_v1/"
//                         ) ? "true" : "false"

//                     env.BUILD_TOKENIZER =
//                         changedFiles.contains(
//                             "api_discovery/tokenizer/"
//                         ) ? "true" : "false"

//                     echo "BUILD_API=${env.BUILD_API}"
//                     echo "BUILD_TOKENIZER=${env.BUILD_TOKENIZER}"

//                     if (
//                         env.BUILD_API != "true" &&
//                         env.BUILD_TOKENIZER != "true"
//                     ) {
//                         error(
//                             "No API or Tokenizer changes detected. " +
//                             "Nothing to deploy."
//                         )
//                     }
//                 }
//             }
//         }

        // stage('Docker Login') {
        //     when {
        //         expression {
        //             env.BUILD_API == "true" ||
        //             env.BUILD_TOKENIZER == "true"
        //         }
        //     }

        //     steps {
        //         withCredentials([
        //             usernamePassword(
        //                 credentialsId: DOCKER_CREDENTIALS_ID,
        //                 usernameVariable: 'DOCKER_USER',
        //                 passwordVariable: 'DOCKER_PASSWORD'
        //             )
        //         ]) {
        //             sh '''
        //                 set -e

        //                 echo "$DOCKER_PASSWORD" |
        //                     docker login \
        //                     --username "$DOCKER_USER" \
        //                     --password-stdin
        //             '''
        //         }
        //     }
        // }

        // stage('Build & Push API') {
        //     when {
        //         expression {
        //             env.BUILD_API == "true"
        //         }
        //     }

        //     steps {
        //         script {
        //             env.API_IMAGE =
        //                 "${DOCKER_REPO}:api-${IMAGE_TAG}"
        //         }

        //         sh '''
        //             set -e

        //             echo "Building API image:"
        //             echo "$API_IMAGE"

        //             docker build \
        //                 --pull \
        //                 -t "$API_IMAGE" \
        //                 api_discovery/api_discovery_v1/

        //             docker push "$API_IMAGE"
        //         '''
        //     }
        // }

        // stage('Build & Push Tokenizer') {
        //     when {
        //         expression {
        //             env.BUILD_TOKENIZER == "true"
        //         }
        //     }

        //     steps {
        //         script {
        //             env.TOKENIZER_IMAGE =
        //                 "${DOCKER_REPO}:tokenizer-${IMAGE_TAG}"
        //         }

        //         withCredentials([
        //             string(
        //                 credentialsId: 'Access-key',
        //                 variable: 'AWS_ACCESS_KEY_ID'
        //             ),
        //             string(
        //                 credentialsId: 'Secret-access-key',
        //                 variable: 'AWS_SECRET_ACCESS_KEY'
        //             )
        //         ]) {

        //             sh '''
        //                 set -e

        //                 echo "Building Tokenizer image:"
        //                 echo "$TOKENIZER_IMAGE"

        //                 docker build \
        //                     --pull \
        //                     --build-arg AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID" \
        //                     --build-arg AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY" \
        //                     --build-arg AWS_DEFAULT_REGION="$AWS_REGION" \
        //                     -t "$TOKENIZER_IMAGE" \
        //                     api_discovery/tokenizer/

        //                 docker push "$TOKENIZER_IMAGE"
        //             '''
        //         }
        //     }
        // }
    }
}