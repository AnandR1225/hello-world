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

        STAGING_NAMESPACE     = "ml-api-discovery-staging"
        PROD_NAMESPACE        = "ml-api-discovery-prod"
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

        stage('Validate Tag') {
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
                            "This pipeline only runs from Git tags."
                        )
                    }
                    if (tagName ==~ /^v\d+\.\d+\.\d+-rc\d+$/) {

                        env.DEPLOY_ENV = "staging"
                        env.NAMESPACE = env.STAGING_NAMESPACE
                        env.SOURCE_BRANCH = "staging"
                    } else if (tagName ==~ /^v\d+\.\d+\.\d+$/) {

                        env.DEPLOY_ENV = "production"
                        env.NAMESPACE = env.PROD_NAMESPACE
                        env.SOURCE_BRANCH = "main"

                    } else {

                        error(
                            "Invalid tag '${tagName}'. " +
                            "Supported formats: " +
                            "vX.Y.Z or vX.Y.Z-rcN"
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
RELEASE VALIDATION
==========================================

Tag:
${env.RELEASE_TAG}

Commit:
${env.TAG_COMMIT}

Environment:
${env.DEPLOY_ENV}

Source Branch:
${env.SOURCE_BRANCH}

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

                    /*
                     * Fetch the correct source branch.
                     *
                     * RC tag:
                     *     staging
                     *
                     * Production tag:
                     *     main
                     */
                    sh """
                        set -e

                        git fetch origin \
                            ${env.SOURCE_BRANCH}:refs/remotes/origin/${env.SOURCE_BRANCH}
                    """

                    def sourceCommit = sh(
                        script: "git rev-parse origin/${env.SOURCE_BRANCH}",
                        returnStdout: true
                    ).trim()

                    echo "Source branch: ${env.SOURCE_BRANCH}"
                    echo "Source HEAD:   ${sourceCommit}"
                    echo "Tagged commit: ${env.TAG_COMMIT}"

                    /*
                     * Verify that the tagged commit exists
                     * in the correct branch history.
                     */
                    def result = sh(
                        script: """
                            git merge-base \
                                --is-ancestor \
                                ${env.TAG_COMMIT} \
                                origin/${env.SOURCE_BRANCH}
                        """,
                        returnStatus: true
                    )

                    if (result != 0) {

                        error(
                            "Tag ${env.RELEASE_TAG} does not belong " +
                            "to the ${env.SOURCE_BRANCH} branch history. " +
                            "Deployment stopped."
                        )
                    }

                    echo """
==========================================
SOURCE VALIDATION PASSED
==========================================

Tag:
${env.RELEASE_TAG}

Environment:
${env.DEPLOY_ENV}

Source Branch:
${env.SOURCE_BRANCH}

Tagged Commit:
${env.TAG_COMMIT}

Branch HEAD:
${sourceCommit}

==========================================
"""
                }
            }
        }

        stage('Docker Login') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "$DOCKER_PASSWORD" |
                            docker login \
                            --username "$DOCKER_USER" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Build & Push API') {
            steps {

                script {
                    env.API_IMAGE =
                        "${DOCKER_REPO}:api-${IMAGE_TAG}"
                }

                sh '''
                    set -e

                    echo "Building API:"
                    echo "$API_IMAGE"

                    docker build \
                        --pull \
                        -t "$API_IMAGE" \
                        api_discovery/api_discovery_v1/

                    docker push "$API_IMAGE"
                '''
            }
        }

        stage('Build & Push Tokenizer') {
            steps {

                script {
                    env.TOKENIZER_IMAGE =
                        "${DOCKER_REPO}:tokenizer-${IMAGE_TAG}"
                }

                withCredentials([
                    string(
                        credentialsId: 'Access-key',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'Secret-access-key',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "Building Tokenizer:"
                        echo "$TOKENIZER_IMAGE"

                        docker build \
                            --pull \
                            --build-arg AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID" \
                            --build-arg AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY" \
                            --build-arg AWS_DEFAULT_REGION="ap-south-1" \
                            -t "$TOKENIZER_IMAGE" \
                            api_discovery/tokenizer/

                        docker push "$TOKENIZER_IMAGE"
                    '''
                }
            }
        }

        stage('Deploy API') {
            steps {

                withKubeConfig(credentialsId: KUBE_CRED) {

                    sh '''
                        set -e

                        echo "Deploying API to:"
                        echo "$NAMESPACE"

                        kubectl apply \
                            -f api_discovery/api_discovery_v1/deployment/fastapi.yaml \
                            -n "$NAMESPACE"

                        kubectl apply \
                            -f api_discovery/api_discovery_v1/deployment/celery.yaml \
                            -n "$NAMESPACE"

                        kubectl set image \
                            deployment/fastapi-app \
                            fastapi="$API_IMAGE" \
                            -n "$NAMESPACE"

                        kubectl set image \
                            deployment/celery-worker \
                            celery="$API_IMAGE" \
                            -n "$NAMESPACE"

                        kubectl rollout status \
                            deployment/fastapi-app \
                            -n "$NAMESPACE" \
                            --timeout=5m

                        kubectl rollout status \
                            deployment/celery-worker \
                            -n "$NAMESPACE" \
                            --timeout=5m
                    '''
                }
            }
        }

        stage('Deploy Tokenizer') {
            steps {

                withKubeConfig(credentialsId: KUBE_CRED) {

                    sh '''
                        set -e

                        echo "Deploying Tokenizer to:"
                        echo "$NAMESPACE"

                        kubectl apply \
                            -f api_discovery/tokenizer/deployment/tokenizer.yaml \
                            -n "$NAMESPACE"

                        kubectl set image \
                            deployment/tokenizer \
                            tokenizer="$TOKENIZER_IMAGE" \
                            -n "$NAMESPACE"

                        kubectl rollout status \
                            deployment/tokenizer \
                            -n "$NAMESPACE" \
                            --timeout=5m
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {

                withKubeConfig(credentialsId: KUBE_CRED) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "DEPLOYMENT VERIFICATION"
                        echo "=========================================="

                        echo "Environment: $DEPLOY_ENV"
                        echo "Branch:      $SOURCE_BRANCH"
                        echo "Tag:         $RELEASE_TAG"
                        echo "Namespace:   $NAMESPACE"

                        echo ""
                        echo "Deployments:"
                        kubectl get deployments -n "$NAMESPACE"

                        echo ""
                        echo "Pods:"
                        kubectl get pods -n "$NAMESPACE"

                        echo "=========================================="
                    '''
                }
            }
        }
    }

    post {

        success {
            echo """
==========================================
DEPLOYMENT SUCCESSFUL
==========================================

Tag:
${env.RELEASE_TAG ?: 'N/A'}

Environment:
${env.DEPLOY_ENV ?: 'N/A'}

Source Branch:
${env.SOURCE_BRANCH ?: 'N/A'}

Namespace:
${env.NAMESPACE ?: 'N/A'}

API Image:
${env.API_IMAGE ?: 'N/A'}

Tokenizer Image:
${env.TOKENIZER_IMAGE ?: 'N/A'}

Build:
${env.BUILD_NUMBER}

==========================================
"""
        }

        failure {
            echo """
==========================================
DEPLOYMENT FAILED
==========================================

Tag:
${env.RELEASE_TAG ?: 'N/A'}

Environment:
${env.DEPLOY_ENV ?: 'N/A'}

Source Branch:
${env.SOURCE_BRANCH ?: 'N/A'}

Build:
${env.BUILD_NUMBER}

URL:
${env.BUILD_URL}

==========================================
"""
        }

        always {
            sh 'docker logout 2>/dev/null || true'
            cleanWs()
        }
    }
}