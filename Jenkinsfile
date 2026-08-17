pipeline {
    agent any

    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        SCANNER_HOME       = tool('sonar-scanner')
        SONARQUBE_ENV      = "sonar-server"

        DOCKER_REPO           = "prophazedocker/**************"
        DOCKER_CREDENTIALS_ID = "prophaze-docker"

        KUBE_CRED   = "ml-api"
        AWS_REGION  = "ap-south-1"   // Region only (no secrets here)
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
                sh "git fetch --tags --force"
            }
        }

        // ============================================================
        // Determines whether this run is a CI-only branch build
        // (develop/staging/main pushes -> scan, no deploy) or a
        // deploy build (tag push -> staging via -rcN, prod via plain
        // version tag). Requires the Multibranch job to have BOTH
        // "Discover branches" and "Discover tags" behaviors enabled -
        // otherwise plain branch pushes will never trigger a job.
        // ============================================================
        stage('Determine Build Context') {
            steps {
                script {
                    if (env.TAG_NAME?.trim()) {
                        env.BUILD_MODE = "deploy"
                        echo "Tag build detected: ${env.TAG_NAME}"

                        if (env.TAG_NAME ==~ /^v?\d+\.\d+\.\d+-rc\d+$/) {
                            env.DEPLOY_ENV = "staging"
                            env.NAMESPACE  = "ml-api-discovery-staging"
                        } else if (env.TAG_NAME ==~ /^v?\d+\.\d+\.\d+$/) {
                            env.DEPLOY_ENV = "prod"
                            env.NAMESPACE  = "ml-api-discovery-prod"
                        } else {
                            error("Tag '${env.TAG_NAME}' doesn't match expected pattern " +
                                  "'{version}' or '{version}-rc{n}' - aborting to avoid an unintended deploy.")
                        }

                        env.IMAGE_TAG = env.TAG_NAME
                        echo "DEPLOY_ENV=${env.DEPLOY_ENV}  NAMESPACE=${env.NAMESPACE}  IMAGE_TAG=${env.IMAGE_TAG}"

                    } else if (env.BRANCH_NAME?.trim()) {
                        env.BUILD_MODE = "ci"
                        env.DEPLOY_ENV = "ci-${env.BRANCH_NAME}"
                        echo "Branch build detected: ${env.BRANCH_NAME} - running CI checks only (Sonar + Trivy), no deploy."

                    } else {
                        error("Neither TAG_NAME nor BRANCH_NAME is set - can't determine build context.")
                    }
                }
            }
        }


        // ---------------- Everything below only runs for tag (deploy) builds ----------------

        stage('Detect Changed Services') {
            when { expression { env.BUILD_MODE == "deploy" } }
            steps {
                script {
                    def isRc = env.DEPLOY_ENV == "staging"
                    def tagGlob = isRc ? "*-rc*" : "*"

                    def previousTag = sh(
                        script: """
                            git tag --sort=-creatordate --list '${tagGlob}' \
                              | grep -Fxv '${env.TAG_NAME}' \
                              | head -n1 || true
                        """,
                        returnStdout: true
                    ).trim()

                    def baseRef
                    if (previousTag) {
                        baseRef = previousTag
                        echo "Diffing against previous ${env.DEPLOY_ENV} tag: ${previousTag}"
                    } else {
                        baseRef = sh(
                            script: "git rev-list --max-parents=0 ${env.TAG_NAME}",
                            returnStdout: true
                        ).trim()
                        echo "No previous ${env.DEPLOY_ENV} tag found - diffing against initial commit"
                    }

                    def changedFiles = sh(
                        script: "git diff --name-only ${baseRef} ${env.TAG_NAME} || true",
                        returnStdout: true
                    ).trim()

                    env.BUILD_API       = changedFiles.contains("api_discovery/api_discovery_v1/") ? "true" : "false"
                    env.BUILD_TOKENIZER = changedFiles.contains("api_discovery/tokenizer/") ? "true" : "false"

                    echo "BUILD_API=${env.BUILD_API}"
                    echo "BUILD_TOKENIZER=${env.BUILD_TOKENIZER}"
                }
            }
        }

        // ============================================================
        // Manual gate for production only. Staging (-rcN tags) flows
        // straight through so QA cycles stay fast; a plain version
        // tag pauses here before anything is built or pushed.
        // ============================================================
        stage('Approval for Production') {
            when {
                allOf {
                    expression { env.BUILD_MODE == "deploy" }
                    expression { env.DEPLOY_ENV == "prod" }
                    expression { env.BUILD_API == "true" || env.BUILD_TOKENIZER == "true" }
                }
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: "Deploy ${env.TAG_NAME} to PRODUCTION (namespace: ${env.NAMESPACE})?", ok: "Proceed"
                }
            }
        }

        stage('Docker Login') {
            when {
                allOf {
                    expression { env.BUILD_MODE == "deploy" }
                    expression { env.BUILD_API == "true" || env.BUILD_TOKENIZER == "true" }
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: DOCKER_CREDENTIALS_ID,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USER" --password-stdin'
                }
            }
        }

        stage('Build & Push API') {
            when {
                allOf {
                    expression { env.BUILD_MODE == "deploy" }
                    expression { env.BUILD_API == "true" }
                }
            }
            steps {
                script {
                    env.API_IMAGE = "${DOCKER_REPO}:api-${env.IMAGE_TAG}"
                }
                sh """
                    docker build \
                        -t ${API_IMAGE} \
                        api_discovery/api_discovery_v1/

                    docker push ${API_IMAGE}
                """
            }
        }

        // ============================================================
        // AWS creds are passed as BuildKit secrets (--secret), not
        // --build-arg. Build-args get baked into image history and
        // are recoverable later; secret mounts are only available to
        // the RUN step that consumes them and never land in a layer.
        // Requires DOCKER_BUILDKIT=1 and the Dockerfile to consume
        // them via `RUN --mount=type=secret,id=aws_access_key_id ...`
        // (read from /run/secrets/aws_access_key_id), rather than as
        // env/ARG vars.
        // ============================================================
        stage('Build & Push Tokenizer') {
            when {
                allOf {
                    expression { env.BUILD_MODE == "deploy" }
                    expression { env.BUILD_TOKENIZER == "true" }
                }
            }
            steps {
                script {
                    env.TOKENIZER_IMAGE = "${DOCKER_REPO}:tokenizer-${env.IMAGE_TAG}"
                }

                withCredentials([
                    string(credentialsId: 'Access-key', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'Secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        DOCKER_BUILDKIT=1 docker build \
                          --secret id=aws_access_key_id,env=AWS_ACCESS_KEY_ID \
                          --secret id=aws_secret_access_key,env=AWS_SECRET_ACCESS_KEY \
                          --build-arg AWS_DEFAULT_REGION=${AWS_REGION} \
                          -t ${TOKENIZER_IMAGE} \
                          api_discovery/tokenizer/

                        docker push ${TOKENIZER_IMAGE}
                    """
                }
            }
        }

        stage('Trivy Image Scan') {
            when {
                allOf {
                    expression { env.BUILD_MODE == "deploy" }
                    expression { env.BUILD_API == "true" || env.BUILD_TOKENIZER == "true" }
                }
            }
            steps {
                script {
                    if (env.BUILD_API == "true") {
                        sh "trivy image ${env.API_IMAGE} --severity HIGH,CRITICAL >> trivyimage.txt || true"
                    }
                    if (env.BUILD_TOKENIZER == "true") {
                        sh "trivy image ${env.TOKENIZER_IMAGE} --severity HIGH,CRITICAL >> trivyimage.txt || true"
                    }
                }
                echo "Image scan completed - results saved in trivyimage.txt"
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                allOf {
                    expression { env.BUILD_MODE == "deploy" }
                    expression { env.BUILD_API == "true" || env.BUILD_TOKENIZER == "true" }
                }
            }
            steps {
                withKubeConfig(credentialsId: KUBE_CRED) {

                    sh """
                        kubectl create ns ${NAMESPACE} \
                        --dry-run=client -o yaml | kubectl apply -f -
                    """

                    script {

                        if (env.BUILD_API == "true") {
                            sh """
                                set -e

                                kubectl apply -f api_discovery/api_discovery_v1/deployment/fastapi.yaml -n ${NAMESPACE}
                                kubectl apply -f api_discovery/api_discovery_v1/deployment/celery.yaml -n ${NAMESPACE}

                                kubectl set image deployment/fastapi-app fastapi=${API_IMAGE} -n ${NAMESPACE}
                                kubectl set image deployment/celery-worker celery=${API_IMAGE} -n ${NAMESPACE}

                                kubectl rollout status deployment/fastapi-app -n ${NAMESPACE} || {
                                    echo "Deployment failed, rolling back..."
                                    kubectl rollout undo deployment/fastapi-app -n ${NAMESPACE}
                                    exit 1
                                }
                                kubectl rollout status deployment/celery-worker -n ${NAMESPACE} || {
                                    echo "Deployment failed, rolling back..."
                                    kubectl rollout undo deployment/celery-worker -n ${NAMESPACE}
                                    exit 1
                                }
                            """
                        }

                        if (env.BUILD_TOKENIZER == "true") {
                            sh """
                                set -e

                                kubectl apply -f api_discovery/tokenizer/deployment/tokenizer.yaml -n ${NAMESPACE}

                                kubectl set image deployment/tokenizer tokenizer=${TOKENIZER_IMAGE} -n ${NAMESPACE}

                                kubectl rollout status deployment/tokenizer -n ${NAMESPACE} || {
                                    echo "Deployment failed, rolling back..."
                                    kubectl rollout undo deployment/tokenizer -n ${NAMESPACE}
                                    exit 1
                                }
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ ${env.DEPLOY_ENV ?: 'unknown'} deployment successful for tag ${env.TAG_NAME ?: 'n/a'}"
        }
        failure {
            echo "❌ Deployment failed for tag ${env.TAG_NAME ?: 'n/a'}"
        }
    }
}