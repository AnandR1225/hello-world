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

        // Uncomment these when you enable the actual deployment
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

        stage('Checkout Tag') {
            steps {
                checkout scm

                sh '''
                    echo "===== Git Information ====="
                    git status
                    git branch --show-current
                    git log -1 --oneline
                    echo "==========================="
                '''
            }
        }

        stage('Validate Tag') {
            steps {
                script {

                    def tagName = env.TAG_NAME?.trim()

                    if (!tagName) {
                        tagName = sh(
                            script: "git describe --tags --exact-match HEAD 2>/dev/null || true",
                            returnStdout: true
                        ).trim()
                    }

                    if (!tagName) {
                        error("No Git tag found. This pipeline is intended to run from a v* tag.")
                    }

                    if (!(tagName ==~ /^v.*/)) {
                        error("Invalid tag '${tagName}'. Expected a tag starting with 'v'.")
                    }

                    env.RELEASE_TAG = tagName

                    echo "Git Release Tag: ${env.RELEASE_TAG}"
                }
            }
        }

        stage('Environment Info') {
            steps {
                echo """
                ==============================
                Environment Information
                ==============================

                Tag:               ${env.RELEASE_TAG}
                Deploy Environment: ${env.DEPLOY_ENV ?: 'Not configured'}
                Repository:         ${env.GIT_REPO}
                Image:              ${env.IMAGE_NAME ?: 'Not configured'}
                Namespace:          ${env.NAMESPACE ?: 'Not configured'}
                Deployment File:    ${env.DEPLOYMENT_FILE ?: 'Not configured'}
                Deployment Name:    ${env.DEPLOYMENT_NAME ?: 'Not configured'}

                ==============================
                """
            }
        }

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

                        echo "Rollback requested."
                        echo "Rollback Docker Tag: ${imageTag}"

                    } else {

                        imageTag = env.RELEASE_TAG

                        echo "Normal release deployment."
                        echo "Release Docker Tag: ${imageTag}"
                    }

                    env.IMAGE_TAG = imageTag

                    echo "FINAL Docker Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Build') {
            when {
                not {
                    expression {
                        params.ROLLBACK
                    }
                }
            }

            steps {
                echo "Building Docker image with tag: ${env.IMAGE_TAG}"

                // Add your Docker build command here
                // sh "docker build -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} ."
            }
        }

        stage('Test') {
            steps {
                echo "Running tests for tag: ${env.RELEASE_TAG}"

                // Add your test commands here
                // sh 'make check'
            }
        }

        stage('Deploy Production') {
            when {
                tag "v*"
            }

            steps {
                echo "=========================================="
                echo "Deploying Production"
                echo "Git Tag:     ${env.RELEASE_TAG}"
                echo "Docker Tag:  ${env.IMAGE_TAG}"
                echo "=========================================="

                // Add your deployment commands here

                // Example:
                // sh """
                //     kubectl -n ${env.NAMESPACE} \
                //       set image deployment/${env.DEPLOYMENT_NAME} \
                //       ${env.DEPLOYMENT_NAME}=${env.IMAGE_NAME}:${env.IMAGE_TAG}
                // """
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
            echo "Release Tag: ${env.RELEASE_TAG}"
            echo "Docker Tag:  ${env.IMAGE_TAG}"
        }

        failure {
            echo "Pipeline failed."
        }

        always {
            echo "Cleaning workspace..."
            cleanWs()
        }
    }
}