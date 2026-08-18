pipeline {
    agent any

    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        GIT_REPO           = "https://github.com/AnandR1225/hello-world.git"
        GIT_CREDENTIALS_ID = "github1"

        // Enable these when you add actual Docker/Kubernetes deployment
        // DOCKER_CREDENTIALS_ID       = "docker-report"
        // SONARQUBE_ENV               = "sonar-server"
        // NAMESPACE                   = "reports"
        // DEPLOY_ENV                  = "production"
        // IMAGE_NAME                  = "prophazedocker/i-report"
        // KUBERNETES_CREDENTIALS_ID   = "k3s-report-staging"
        // DEPLOYMENT_FILE             = "prod-reports.yaml"
        // DEPLOYMENT_NAME             = "prod-reports-api"
    }

    parameters {
        booleanParam(
            name: 'ROLLBACK',
            defaultValue: false,
            description: 'Rollback to TARGET_VERSION instead of deploying the current Git tag'
        )

        string(
            name: 'TARGET_VERSION',
            defaultValue: '',
            description: 'Target Docker tag for rollback, e.g. v1.2.0'
        )
    }

    stages {

        /*
         * ============================================================
         * CLEAN WORKSPACE
         * ============================================================
         */

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        /*
         * ============================================================
         * CHECKOUT
         * ============================================================
         */

        stage('Checkout Tag') {
            steps {
                script {

                    echo "Checking out source using Jenkins SCM configuration..."

                    checkout scm

                    sh '''
                        echo "=========================================="
                        echo "Git Information"
                        echo "=========================================="

                        git status

                        echo ""
                        echo "Current commit:"
                        git log -1 --oneline

                        echo ""
                        echo "Git tags pointing to HEAD:"
                        git tag --points-at HEAD || true

                        echo "=========================================="
                    '''
                }
            }
        }

        /*
         * ============================================================
         * VALIDATE TAG
         * ============================================================
         */

        stage('Validate Tag') {
            steps {
                script {

                    /*
                     * Jenkins may provide TAG_NAME when the build
                     * is triggered as a tag build.
                     *
                     * We still use git describe as a fallback because
                     * it verifies the actual checked-out commit.
                     */

                    def tagName = env.TAG_NAME?.trim()

                    if (!tagName) {
                        tagName = sh(
                            script: '''
                                git describe \
                                --tags \
                                --exact-match \
                                HEAD \
                                2>/dev/null || true
                            ''',
                            returnStdout: true
                        ).trim()
                    }

                    /*
                     * No tag = do not continue.
                     */

                    if (!tagName) {
                        error """
                        No Git tag was found on the checked-out commit.

                        This Jenkins pipeline is configured for
                        Git tag based deployments only.

                        Expected example:
                        v1.0.0
                        """
                    }

                    /*
                     * Only v* tags are allowed.
                     */

                    if (!(tagName ==~ /^v.*/)) {
                        error """
                        Invalid Git tag: '${tagName}'

                        Production tags must start with 'v'.

                        Valid examples:
                        v1.0.0
                        v1.0.1
                        v1.1.0
                        """
                    }

                    /*
                     * Store the validated release tag.
                     */

                    env.RELEASE_TAG = tagName

                    echo """
                    ==========================================
                    TAG VALIDATION SUCCESSFUL
                    ==========================================

                    Git Release Tag: ${env.RELEASE_TAG}

                    ==========================================
                    """
                }
            }
        }

        /*
         * ============================================================
         * ENVIRONMENT INFORMATION
         * ============================================================
         */

        stage('Environment Info') {
            steps {
                echo """
                ==========================================
                Environment Information
                ==========================================

                Repository:
                ${env.GIT_REPO}

                Release Tag:
                ${env.RELEASE_TAG}

                Deploy Environment:
                ${env.DEPLOY_ENV ?: 'Not configured'}

                Docker Image:
                ${env.IMAGE_NAME ?: 'Not configured'}

                Namespace:
                ${env.NAMESPACE ?: 'Not configured'}

                Deployment File:
                ${env.DEPLOYMENT_FILE ?: 'Not configured'}

                Deployment Name:
                ${env.DEPLOYMENT_NAME ?: 'Not configured'}

                ==========================================
                """
            }
        }

        /*
         * ============================================================
         * GENERATE DOCKER TAG
         * ============================================================
         */

        stage('Generate Docker Tag') {
            steps {
                script {

                    def imageTag = ""

                    if (params.ROLLBACK) {

                        if (!params.TARGET_VERSION?.trim()) {
                            error(
                                "Rollback requested but TARGET_VERSION was not provided."
                            )
                        }

                        imageTag = params.TARGET_VERSION.trim()

                        /*
                         * Validate rollback version.
                         */

                        if (!(imageTag ==~ /^v.*/)) {
                            error(
                                "Invalid rollback version '${imageTag}'. " +
                                "Rollback versions must start with 'v'."
                            )
                        }

                        echo """
                        ==========================================
                        ROLLBACK
                        ==========================================

                        Rollback Docker Tag:
                        ${imageTag}

                        ==========================================
                        """

                    } else {

                        imageTag = env.RELEASE_TAG

                        echo """
                        ==========================================
                        RELEASE DEPLOYMENT
                        ==========================================

                        Release Docker Tag:
                        ${imageTag}

                        ==========================================
                        """
                    }

                    env.IMAGE_TAG = imageTag

                    echo "FINAL Docker Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        /*
         * ============================================================
         * BUILD
         * ============================================================
         */

        stage('Build') {
            when {
                expression {
                    return !params.ROLLBACK
                }
            }

            steps {
                script {

                    echo """
                    ==========================================
                    BUILD
                    ==========================================

                    Git Tag:
                    ${env.RELEASE_TAG}

                    Docker Tag:
                    ${env.IMAGE_TAG}

                    ==========================================
                    """

                    /*
                     * Add your actual build command here.
                     *
                     * Example:
                     *
                     * sh """
                     *     docker build \
                     *       -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} .
                     * """
                     */

                    echo "Build stage completed."
                }
            }
        }

        /*
         * ============================================================
         * TEST
         * ============================================================
         */

        stage('Test') {
            steps {

                echo """
                ==========================================
                TEST
                ==========================================

                Testing release:
                ${env.RELEASE_TAG}

                ==========================================
                """

                /*
                 * Add your test commands here.
                 *
                 * Example:
                 *
                 * sh 'make check'
                 */

                echo "Test stage completed."
            }
        }

        /*
         * ============================================================
         * PRODUCTION DEPLOYMENT
         * ============================================================
         */

        stage('Deploy Production') {

            when {
                tag "v*"
            }

            steps {
                script {

                    echo """
                    ==========================================
                    PRODUCTION DEPLOYMENT
                    ==========================================

                    Git Tag:
                    ${env.RELEASE_TAG}

                    Docker Tag:
                    ${env.IMAGE_TAG}

                    Environment:
                    ${env.DEPLOY_ENV ?: 'Not configured'}

                    Namespace:
                    ${env.NAMESPACE ?: 'Not configured'}

                    ==========================================
                    """

                    /*
                     * Add your actual deployment commands here.
                     *
                     * Example:
                     *
                     * withKubeConfig(
                     *     credentialsId:
                     *         env.KUBERNETES_CREDENTIALS_ID
                     * ) {
                     *
                     *     sh """
                     *         kubectl -n ${env.NAMESPACE} \
                     *           set image \
                     *           deployment/${env.DEPLOYMENT_NAME} \
                     *           ${env.DEPLOYMENT_NAME}=\
                     *           ${env.IMAGE_NAME}:${env.IMAGE_TAG}
                     *
                     *         kubectl -n ${env.NAMESPACE} \
                     *           rollout status \
                     *           deployment/${env.DEPLOYMENT_NAME}
                     *     """
                     * }
                     */

                    echo "Production deployment stage completed."
                }
            }
        }
    }
}