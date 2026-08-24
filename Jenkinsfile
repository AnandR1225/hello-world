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

        NAMESPACE             = "ml-api-discovery-staging"
        SOURCE_BRANCH         = "staging"
        AWS_REGION            = "ap-south-1"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    set -e

                    echo "======================================"
                    echo "Commit:"
                    git log -1 --oneline

                    echo "Commit SHA:"
                    git rev-parse HEAD

                    echo "Current Branch:"
                    git branch --show-current || true

                    echo "Tags on HEAD:"
                    git tag --points-at HEAD || true

                    echo "======================================"
                '''
            }
        }

        stage('Detect Build Type') {
            steps {
                script {

                    def tagName = sh(
                        script: 'git tag --points-at HEAD | head -n 1',
                        returnStdout: true
                    ).trim()

                    env.RELEASE_TAG = ""
                    env.IMAGE_TAG = ""
                    env.TAG_COMMIT = ""

                    if (tagName) {

                        echo "Tag detected: ${tagName}"

                        if (!(tagName ==~ /^v\\d+\\.\\d+\\.\\d+-rc\\d+$/)) {
                            error(
                                "Invalid tag '${tagName}'. " +
                                "Expected format: vX.Y.Z-rcN, e.g. v1.2.22-rc1"
                            )
                        }

                        env.RELEASE_TAG = tagName
                        env.IMAGE_TAG = tagName

                        env.TAG_COMMIT = sh(
                            script: 'git rev-parse HEAD',
                            returnStdout: true
                        ).trim()

                        env.BUILD_TYPE = "TAG"

                        echo "======================================"
                        echo "Build Type : TAG"
                        echo "Release Tag: ${env.RELEASE_TAG}"
                        echo "Tag Commit : ${env.TAG_COMMIT}"
                        echo "======================================"

                    } else {

                        env.BUILD_TYPE = "STAGING"

                        echo "======================================"
                        echo "Build Type : STAGING"
                        echo "No release tag found on HEAD."
                        echo "======================================"
                    }
                }
            }
        }

        stage('Validate Staging Branch') {
            when {
                expression {
                    return env.BUILD_TYPE == "TAG"
                }
            }

            steps {
                script {

                    sh '''
                        set -e

                        echo "Fetching latest staging branch..."

                        git fetch origin \
                            staging:refs/remotes/origin/staging
                    '''

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
                            "Tag ${env.RELEASE_TAG} does not belong " +
                            "to the staging branch. Deployment stopped."
                        )
                    }

                    echo "======================================"
                    echo "Tag Validation Successful"
                    echo "Tag ${env.RELEASE_TAG} belongs to staging."
                    echo "======================================"
                }
            }
        }

        stage('Set Image Tag') {
            steps {
                script {

                    if (env.BUILD_TYPE == "TAG") {

                        env.IMAGE_TAG = env.RELEASE_TAG

                    } else {

                        def commitSha = sh(
                            script: 'git rev-parse --short HEAD',
                            returnStdout: true
                        ).trim()

                        env.IMAGE_TAG = "staging-${commitSha}"
                    }

                    echo "Docker Image Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        /*
        ============================================================
        Docker Login
        ============================================================
        */

        // stage('Docker Login') {
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
        //
        //                 echo "$DOCKER_PASSWORD" |
        //                 docker login \
        //                     --username "$DOCKER_USER" \
        //                     --password-stdin
        //             '''
        //         }
        //     }
        // }

        /*
        ============================================================
        Build & Push API
        ============================================================
        */

        // stage('Build & Push API') {
        //     steps {
        //         script {
        //             env.API_IMAGE =
        //                 "${DOCKER_REPO}:api-${IMAGE_TAG}"
        //         }
        //
        //         sh '''
        //             set -e
        //
        //             docker build \
        //                 --pull \
        //                 -t "$API_IMAGE" \
        //                 api_discovery/api_discovery_v1/
        //
        //             docker push "$API_IMAGE"
        //         '''
        //     }
        // }

        /*
        ============================================================
        Build & Push Tokenizer
        ============================================================
        */

        // stage('Build & Push Tokenizer') {
        //     steps {
        //         script {
        //             env.TOKENIZER_IMAGE =
        //                 "${DOCKER_REPO}:tokenizer-${IMAGE_TAG}"
        //         }
        //
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
        //
        //                 docker build \
        //                     --pull \
        //                     --build-arg AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID" \
        //                     --build-arg AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY" \
        //                     --build-arg AWS_DEFAULT_REGION="$AWS_REGION" \
        //                     -t "$TOKENIZER_IMAGE" \
        //                     api_discovery/tokenizer/
        //
        //                 docker push "$TOKENIZER_IMAGE"
        //             '''
        //         }
        //     }
        // }

        /*
        ============================================================
        Deploy API
        ============================================================
        */

        // stage('Deploy API') {
        //     steps {
        //         withKubeConfig(credentialsId: KUBE_CRED) {
        //             sh '''
        //                 set -e
        //
        //                 kubectl apply \
        //                     -f api_discovery/api_discovery_v1/deployment/fastapi.yaml \
        //                     -n "$NAMESPACE"
        //
        //                 kubectl apply \
        //                     -f api_discovery/api_discovery_v1/deployment/celery.yaml \
        //                     -n "$NAMESPACE"
        //
        //                 kubectl set image \
        //                     deployment/fastapi-app \
        //                     fastapi="$API_IMAGE" \
        //                     -n "$NAMESPACE"
        //
        //                 kubectl set image \
        //                     deployment/celery-worker \
        //                     celery="$API_IMAGE" \
        //                     -n "$NAMESPACE"
        //
        //                 kubectl rollout status \
        //                     deployment/fastapi-app \
        //                     -n "$NAMESPACE" \
        //                     --timeout=5m
        //
        //                 kubectl rollout status \
        //                     deployment/celery-worker \
        //                     -n "$NAMESPACE" \
        //                     --timeout=5m
        //             '''
        //         }
        //     }
        // }

        /*
        ============================================================
        Deploy Tokenizer
        ============================================================
        */

        // stage('Deploy Tokenizer') {
        //     steps {
        //         withKubeConfig(credentialsId: KUBE_CRED) {
        //             sh '''
        //                 set -e
        //
        //                 kubectl apply \
        //                     -f api_discovery/tokenizer/deployment/tokenizer.yaml \
        //                     -n "$NAMESPACE"
        //
        //                 kubectl set image \
        //                     deployment/tokenizer \
        //                     tokenizer="$TOKENIZER_IMAGE" \
        //                     -n "$NAMESPACE"
        //
        //                 kubectl rollout status \
        //                     deployment/tokenizer \
        //                     -n "$NAMESPACE" \
        //                     --timeout=5m
        //             '''
        //         }
        //     }
        // }

        /*
        ============================================================
        Verify Deployment
        ============================================================
        */

        // stage('Verify Deployment') {
        //     steps {
        //         withKubeConfig(credentialsId: KUBE_CRED) {
        //             sh '''
        //                 kubectl get deployments -n "$NAMESPACE"
        //                 kubectl get pods -n "$NAMESPACE"
        //             '''
        //         }
        //     }
        // }
    }

    post {
        success {
            echo "======================================"
            echo "Jenkins build completed successfully."
            echo "Build Type : ${env.BUILD_TYPE}"
            echo "Image Tag  : ${env.IMAGE_TAG}"
            echo "======================================"
        }

        failure {
            echo "======================================"
            echo "Jenkins build failed."
            echo "Build Type : ${env.BUILD_TYPE}"
            echo "======================================"
        }
    }
}