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
         * CHECKOUT TAG
         * ============================================================
         */

        stage('Checkout Tag') {
            steps {
                script {

                    echo "=========================================="
                    echo "Checking out Git tag"
                    echo "=========================================="

                    /*
                     * Jenkins SCM configuration determines which
                     * tag triggered this build.
                     *
                     * Do NOT checkout master/staging manually here.
                     */

                    checkout scm

                    sh '''
                        echo "=========================================="
                        echo "Git Information"
                        echo "=========================================="

                        echo "Commit:"
                        git log -1 --oneline

                        echo ""
                        echo "HEAD:"
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
         * VALIDATE TAG
         * ============================================================
         */

        stage('Validate Tag') {
            steps {
                script {

                    /*
                     * Find the tag pointing to the checked-out commit.
                     *
                     * We intentionally don't depend only on TAG_NAME,
                     * because TAG_NAME may not always be populated
                     * depending on how Jenkins SCM is configured.
                     */

                    def tagName = sh(
                        script: '''
                            git tag --points-at HEAD | head -n 1
                        ''',
                        returnStdout: true
                    ).trim()

                    /*
                     * Fallback
                     */

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
                     * No tag = stop
                     */

                    if (!tagName) {
                        error """
                        ==========================================
                        TAG VALIDATION FAILED
                        ==========================================

                        No Git tag was found on the checked-out commit.

                        This Jenkins pipeline accepts only Git tags.

                        Expected:
                        v1.0.0

                        ==========================================
                        """
                    }

                    /*
                     * Only v* tags are allowed
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
                     * Save release tag
                     */

                    env.RELEASE_TAG = tagName

                    /*
                     * Get tagged commit
                     */

                    env.TAG_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    echo """
                    ==========================================
                    TAG VALIDATION SUCCESSFUL
                    ==========================================

                    Release Tag:
                    ${env.RELEASE_TAG}

                    Tagged Commit:
                    ${env.TAG_COMMIT}

                    ==========================================
                    """
                }
            }
        }

        /*
         * ============================================================
         * VALIDATE STAGING SOURCE
         * ============================================================
         */

        stage('Validate Staging Source') {
            steps {
                script {

                    echo "Fetching staging branch..."

                    /*
                     * Fetch staging only for validation.
                     *
                     * This does NOT change the checked-out tag.
                     */

                    sh '''
                        git fetch origin \
                            staging:refs/remotes/origin/staging
                    '''

                    /*
                     * Get staging HEAD
                     */

                    def stagingCommit = sh(
                        script: 'git rev-parse origin/staging',
                        returnStdout: true
                    ).trim()

                    /*
                     * Save for later stages
                     */

                    env.STAGING_COMMIT = stagingCommit

                    echo """
                    ==========================================
                    STAGING VALIDATION
                    ==========================================

                    Release Tag:
                    ${env.RELEASE_TAG}

                    Tagged Commit:
                    ${env.TAG_COMMIT}

                    Staging HEAD:
                    ${env.STAGING_COMMIT}

                    ==========================================
                    """

                    /*
                     * Check whether the tagged commit exists
                     * in the staging branch history.
                     *
                     * This is better than comparing:
                     *
                     * tagCommit == stagingCommit
                     *
                     * because staging can move forward after
                     * a release tag is created.
                     */

                    def stagingCheck = sh(
                        script: """
                            git merge-base \
                                --is-ancestor \
                                ${env.TAG_COMMIT} \
                                origin/staging
                        """,
                        returnStatus: true
                    )

                    if (stagingCheck != 0) {

                        error """
                        ==========================================
                        INVALID TAG SOURCE
                        ==========================================

                        Release Tag:
                        ${env.RELEASE_TAG}

                        Tagged Commit:
                        ${env.TAG_COMMIT}

                        Staging HEAD:
                        ${env.STAGING_COMMIT}

                        RESULT:
                        The tagged commit does NOT exist in the
                        staging branch history.

                        Only tags created from commits belonging
                        to the staging branch are allowed.

                        ==========================================
                        """
                    }

                    env.SOURCE_BRANCH = "staging"

                    echo """
                    ==========================================
                    STAGING VALIDATION PASSED
                    ==========================================

                    Release Tag:
                    ${env.RELEASE_TAG}

                    Source Branch:
                    ${env.SOURCE_BRANCH}

                    Tagged Commit:
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
                ENVIRONMENT INFORMATION
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

                    env.IMAGE_TAG = env.RELEASE_TAG

                    echo """
                    ==========================================
                    DOCKER TAG
                    ==========================================

                    Git Release Tag:
                    ${env.RELEASE_TAG}

                    Docker Image Tag:
                    ${env.IMAGE_TAG}

                    ==========================================
                    """
                }
            }
        }
            stage('Build') {
            steps {
                script {

                    echo """
                    ==========================================
                    BUILD
                    ==========================================

                    Release:
                    ${env.RELEASE_TAG}

                    Docker Tag:
                    ${env.IMAGE_TAG}

                    ==========================================
                    """

                    /*
                     * Add actual build command here.
                     *
                     * Example:
                     *
                     * sh """
                     *     docker build \
                     *         -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} .
                     * """
                     */

                    echo "Build stage completed."
                }
            }
        }
    }
}

// jenkinsfile testing webhook configured, now i need to test.