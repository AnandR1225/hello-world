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

                    echo "========================================"
                    echo "Git Information"
                    echo "========================================"

                    echo "Commit:"
                    git log -1 --oneline

                    echo ""
                    echo "Commit SHA:"
                    git rev-parse HEAD

                    echo ""
                    echo "Tags pointing to HEAD:"
                    git tag --points-at HEAD || true

                    echo ""
                    echo "Current branch/ref:"
                    git branch -a --show-current || true
                '''
            }
        }

        stage('Validate Staging Tag') {
            steps {
                script {

                    def tagName = sh(
                        script: '''
                            git tag --points-at HEAD | head -n 1
                        ''',
                        returnStdout: true
                    ).trim()

                    if (!tagName) {
                        error(
                            "No Git tag found on HEAD. " +
                            "Staging deployment requires a release candidate tag " +
                            "in the format vX.Y.Z-rcN."
                        )
                    }

                    if (!(tagName ==~ /^v\d+\.\d+\.\d+-rc\d+$/)) {
                        error(
                            "Invalid tag '${tagName}'. " +
                            "Expected format: vX.Y.Z-rcN, " +
                            "for example v1.2.22-rc1."
                        )
                    }

                    env.RELEASE_TAG = tagName
                    env.IMAGE_TAG = tagName

                    env.TAG_COMMIT = sh(
                        script: '''
                            git rev-parse HEAD
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "========================================"
                    echo "Staging Release Information"
                    echo "========================================"
                    echo "Release Tag : ${env.RELEASE_TAG}"
                    echo "Commit SHA   : ${env.TAG_COMMIT}"
                }
            }
        }

        stage('Validate Staging Branch') {
            steps {
                script {

                    sh '''
                        set -e

                        echo "Fetching staging branch..."

                        git fetch origin \
                            staging:refs/remotes/origin/staging

                        echo "Staging branch fetched successfully."
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
                            "Tag ${env.RELEASE_TAG} (${env.TAG_COMMIT}) " +
                            "does not belong to the staging branch. " +
                            "Deployment stopped."
                        )
                    }

                    echo "========================================"
                    echo "Staging Branch Validation"
                    echo "========================================"
                    echo "Tag ${env.RELEASE_TAG} belongs to staging branch."
                }
            }
        }
    }

    post {

        success {
            echo "========================================"
            echo "PIPELINE SUCCESS"
            echo "========================================"

            script {
                if (env.RELEASE_TAG) {
                    echo "Release Tag: ${env.RELEASE_TAG}"
                }
            }
        }

        failure {
            echo "========================================"
            echo "PIPELINE FAILED"
            echo "========================================"

            script {
                if (env.RELEASE_TAG) {
                    echo "Release Tag: ${env.RELEASE_TAG}"
                }
            }
        }

        always {
            echo "Jenkins build completed."
        }
    }
}

// testing with ishanu