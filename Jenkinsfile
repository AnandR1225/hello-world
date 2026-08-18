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

        // Enable these when actual deployment is configured
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

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        /*
         * ============================================================
         * CHECKOUT TAG
         * ============================================================
         */

        stage('Checkout Tag') {
            steps {
                script {

                    echo "Checking out source from Jenkins SCM configuration..."

                    checkout scm

                    sh '''
                        echo "=========================================="
                        echo "Git Information"
                        echo "=========================================="

                        echo "Commit:"
                        git log -1 --oneline

                        echo ""
                        echo "Current HEAD:"
                        git rev-parse HEAD

                        echo ""
                        echo "Tags pointing to HEAD:"
                        git tag --points-at HEAD || true

                        echo "=========================================="
                    '''
                }
            }
        }

        /*
         * ============================================================
         * VALIDATE TAG + STAGING BRANCH
         * ============================================================
         */

        stage('Validate Tag and Staging Source') {
            steps {
                script {

                    /*
                     * Get the Git tag.
                     *
                     * TAG_NAME may be provided by Jenkins.
                     * git describe is used as a fallback.
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
                     * ------------------------------------------------
                     * Validate that a tag exists
                     * ------------------------------------------------
                     */

                    if (!tagName) {
                        error """
                        ==========================================
                        TAG VALIDATION FAILED
                        ==========================================

                        No Git tag was found on the checked-out commit.

                        This Jenkins job only accepts Git tags.

                        Example:
                        v1.0.0

                        ==========================================
                        """
                    }

                    /*
                     * ------------------------------------------------
                     * Validate tag format
                     * ------------------------------------------------
                     */

                    if (!(tagName ==~ /^v.*/)) {
                        error """
                        ==========================================
                        INVALID TAG
                        ==========================================

                        Tag:
                        ${tagName}

                        Only tags beginning with 'v' are allowed.

                        Valid examples:
                        v1.0.0
                        v1.0.1
                        v2.0.0

                        ==========================================
                        """
                    }

                    /*
                     * ------------------------------------------------
                     * Get tagged commit
                     * ------------------------------------------------
                     */

                    def tagCommit = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    echo """
                    Git Tag:
                    ${tagName}

                    Tagged Commit:
                    ${tagCommit}
                    """

                    /*
                     * ------------------------------------------------
                     * Fetch staging branch
                     *
                     * The Jenkins SCM refspec only fetches tags,
                     * so we explicitly fetch staging here.
                     * ------------------------------------------------
                     */

                    sh '''
                        echo "Fetching staging branch..."

                        git fetch origin \
                            staging:refs/remotes/origin/staging
                    '''

                    /*
                     * ------------------------------------------------
                     * Get staging HEAD
                     * ------------------------------------------------
                     */

                    def stagingCommit = sh(
                        script: 'git rev-parse origin/staging',
                        returnStdout: true
                    ).trim()

                    echo """
                    ==========================================
                    STAGING VALIDATION
                    ==========================================

                    Git Tag:
                    ${tagName}

                    Tagged Commit:
                    ${tagCommit}

                    Current Staging HEAD:
                    ${stagingCommit}

                    ==========================================
                    """

                    /*
                     * ------------------------------------------------
                     * Verify that the tagged commit belongs to
                     * the staging branch.
                     *
                     * --is-ancestor means:
                     *
                     * tagged commit
                     *       |
                     *       v
                     * staging history
                     *
                     * Therefore the tag must point to a commit
                     * that exists in staging.
                     * ------------------------------------------------
                     */

                    def isFromStaging = sh(
                        script: """
                            git merge-base \
                                --is-ancestor \
                                ${tagCommit} \
                                origin/staging
                        """,
                        returnStatus: true
                    )

                    if (isFromStaging != 0) {
                        error """
                        ==========================================
                        INVALID TAG SOURCE
                        ==========================================

                        Tag:
                        ${tagName}

                        Tagged Commit:
                        ${tagCommit}

                        Staging HEAD:
                        ${stagingCommit}

                        RESULT:
                        The tagged commit is NOT part of the
                        staging branch.

                        This Jenkins pipeline only allows tags
                        created from the staging branch.

                        ==========================================
                        """
                    }

                    /*
                     * ------------------------------------------------
                     * Validation successful
                     * ------------------------------------------------
                     */

                    env.RELEASE_TAG = tagName
                    env.TAG_COMMIT = tagCommit
                    env.SOURCE_BRANCH = "staging"

                    echo """
                    ==========================================
                    TAG VALIDATION SUCCESSFUL
                    ==========================================

                    Release Tag:
                    ${env.RELEASE_TAG}

                    Source Branch:
                    ${env.SOURCE_BRANCH}

                    Commit:
                    ${env.TAG_COMMIT}

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

                Source Branch:
                ${env.SOURCE_BRANCH}

                Release Tag:
                ${env.RELEASE_TAG}

                Commit:
                ${env.TAG_COMMIT}

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

                    def imageTag

                    if (params.ROLLBACK) {

                        if (!params.TARGET_VERSION?.trim()) {
                            error(
                                "Rollback requested but TARGET_VERSION was not provided."
                            )
                        }

                        imageTag = params.TARGET_VERSION.trim()

                        if (!(imageTag ==~ /^v.*/)) {
                            error(
                                "Invalid rollback version '${imageTag}'. " +
                                "Rollback versions must start with 'v'."
                            )
                        }

                        echo """
                        ==========================================
                        ROLLBACK REQUESTED
                        ==========================================

                        Rollback Version:
                        ${imageTag}

                        ==========================================
                        """

                    } else {

                        imageTag = env.RELEASE_TAG

                        echo """
                        ==========================================
                        RELEASE DEPLOYMENT
                        ==========================================

                        Release Tag:
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

                    Release Tag:
                    ${env.RELEASE_TAG}

                    Docker Tag:
                    ${env.IMAGE_TAG}

                    ==========================================
                    """

                    /*
                     * Add Docker build here later.
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
                script {

                    echo """
                    ==========================================
                    TEST
                    ==========================================

                    Testing Release:
                    ${env.RELEASE_TAG}

                    Source Branch:
                    ${env.SOURCE_BRANCH}

                    ==========================================
                    """

                    /*
                     * Add your tests here.
                     *
                     * Example:
                     *
                     * sh 'make check'
                     */

                    echo "Test stage completed."
                }
            }
        }

        /*
         * ============================================================
         * PRODUCTION DEPLOYMENT
         * ============================================================
         */

        stage('Deploy Production') {
            when {
                allOf {
                    expression {
                        return !params.ROLLBACK
                    }

                    expression {
                        return env.RELEASE_TAG?.startsWith('v')
                    }
                }
            }

            steps {
                script {

                    echo """
                    ==========================================
                    PRODUCTION DEPLOYMENT
                    ==========================================

                    Source Branch:
                    ${env.SOURCE_BRANCH}

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
                     * Add your actual Kubernetes deployment here.
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
                     *         set image \
                     *         deployment/${env.DEPLOYMENT_NAME} \
                     *         ${env.DEPLOYMENT_NAME}=\
                     *         ${env.IMAGE_NAME}:${env.IMAGE_TAG}
                     *
                     *         kubectl -n ${env.NAMESPACE} \
                     *         rollout status \
                     *         deployment/${env.DEPLOYMENT_NAME}
                     *     """
                     * }
                     */

                    echo "Production deployment stage completed."
                }
            }
        }
    }

    /*
     * ================================================================
     * POST ACTIONS
     * ================================================================
     */

    post {

        success {
            echo """
            ==========================================
            PIPELINE SUCCESS
            ==========================================

            Repository:
            ${env.GIT_REPO}

            Source Branch:
            ${env.SOURCE_BRANCH ?: 'N/A'}

            Release Tag:
            ${env.RELEASE_TAG ?: 'N/A'}

            Docker Tag:
            ${env.IMAGE_TAG ?: 'N/A'}

            Build:
            ${env.BUILD_NUMBER}

            ==========================================
            """
        }

        failure {
            echo """
            ==========================================
            PIPELINE FAILED
            ==========================================

            Repository:
            ${env.GIT_REPO}

            Release Tag:
            ${env.RELEASE_TAG ?: 'N/A'}

            Build:
            ${env.BUILD_NUMBER}

            Build URL:
            ${env.BUILD_URL}

            ==========================================
            """
        }

        always {
            echo "Pipeline completed."
            echo "Cleaning workspace..."

            cleanWs()
        }
    }
}