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

                    echo "Commit:"
                    git log -1 --oneline

                    echo "Commit SHA:"
                    git rev-parse HEAD

                    echo "Tag:"
                    git tag --points-at HEAD || true
                '''
            }
        }

        stage('Validate Staging Tag') {
            steps {
                script {
                    def tagName = sh(
                        script: 'git tag --points-at HEAD | head -n 1',
                        returnStdout: true
                    ).trim()

                    if (!tagName) {
                        error(
                            "No Git tag found. " +
                            "Staging deployment requires a release candidate tag."
                        )
                    }

                    if (!(tagName ==~ /^v\d+\.\d+\.\d+-rc\d+$/)) {
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

                    echo "Staging Release Tag: ${env.RELEASE_TAG}"
                }
            }
        }

        stage('Validate Staging Branch') {
            steps {
                script {
                    sh '''
                        set -e

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

                    echo "Tag belongs to staging branch."
                }
            }
        }

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
        //                 echo "$DOCKER_PASSWORD" |
        //                 docker login \
        //                     --username "$DOCKER_USER" \
        //                     --password-stdin
        //             '''
        //         }
        //     }
        // }

        // stage('Build & Push API') {
        //     steps {
        //         script {
        //             env.API_IMAGE =
        //                 "${DOCKER_REPO}:api-${IMAGE_TAG}"
        //         }

        //         sh '''
        //             set -e

        //             docker build \
        //                 --pull \
        //                 -t "$API_IMAGE" \
        //                 api_discovery/api_discovery_v1/

        //             docker push "$API_IMAGE"
        //         '''
        //     }
        // }

        // stage('Build & Push Tokenizer') {
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

        // stage('Deploy API') {
        //     steps {
        //         withKubeConfig(credentialsId: KUBE_CRED) {
        //             sh '''
        //                 set -e

        //                 kubectl apply \
        //                     -f api_discovery/api_discovery_v1/deployment/fastapi.yaml \
        //                     -n "$NAMESPACE"

        //                 kubectl apply \
        //                     -f api_discovery/api_discovery_v1/deployment/celery.yaml \
        //                     -n "$NAMESPACE"

        //                 kubectl set image \
        //                     deployment/fastapi-app \
        //                     fastapi="$API_IMAGE" \
        //                     -n "$NAMESPACE"

        //                 kubectl set image \
        //                     deployment/celery-worker \
        //                     celery="$API_IMAGE" \
        //                     -n "$NAMESPACE"

        //                 kubectl rollout status \
        //                     deployment/fastapi-app \
        //                     -n "$NAMESPACE" \
        //                     --timeout=5m

        //                 kubectl rollout status \
        //                     deployment/celery-worker \
        //                     -n "$NAMESPACE" \
        //                     --timeout=5m
        //             '''
        //         }
        //     }
        // }

        // stage('Deploy Tokenizer') {
        //     steps {
        //         withKubeConfig(credentialsId: KUBE_CRED) {
        //             sh '''
        //                 set -e

        //                 kubectl apply \
        //                     -f api_discovery/tokenizer/deployment/tokenizer.yaml \
        //                     -n "$NAMESPACE"

        //                 kubectl set image \
        //                     deployment/tokenizer \
        //                     tokenizer="$TOKENIZER_IMAGE" \
        //                     -n "$NAMESPACE"

        //                 kubectl rollout status \
        //                     deployment/tokenizer \
        //                     -n "$NAMESPACE" \
        //                     --timeout=5m
        //             '''
        //         }
        //     }
        // }

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

}