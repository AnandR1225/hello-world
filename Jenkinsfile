pipeline {
    agent any

    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        // DOCKER_REPO = "prophazedocker/ml-api-discovery"
        // DOCKER_CREDENTIALS_ID = "prophaze-docker"
        // KUBE_CRED = "ml-api"
        // AWS_REGION = "ap-south-1"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'git fetch --tags --force'
            }
        }

        stage('Validate Build Type') {
            steps {
                script {

                    if (env.TAG_NAME?.trim()) {

                        echo "Tag detected: ${env.TAG_NAME}"

                        if (!(env.TAG_NAME ==~ /^v\d+\.\d+\.\d+$/)) {
                            error(
                                "Invalid production tag: ${env.TAG_NAME}. " +
                                "Expected format: v1.0.0"
                            )
                        }

                        env.BUILD_MODE = 'production'
                        env.IMAGE_TAG = env.TAG_NAME
                        env.NAMESPACE = 'ml-api-discovery-prod'

                        echo """
                        ================================
                        PRODUCTION RELEASE
                        Tag       : ${env.TAG_NAME}
                        Image Tag : ${env.IMAGE_TAG}
                        Namespace : ${env.NAMESPACE}
                        ================================
                        """

                    } else {

                        env.BUILD_MODE = 'ci'

                        echo """
                        ==================================
                        CI BUILD ONLY
                        Branch: ${env.BRANCH_NAME}
                        NO PRODUCTION DEPLOYMENT
                        ==================================
                        """
                    }
                }
            }
        }

        stage('SonarQube') {
            when {
                expression {
                    env.BUILD_MODE == 'ci' ||
                    env.BUILD_MODE == 'production'
                }
            }

            steps {
                echo 'Running SonarQube scan...'

                // Your SonarQube commands
            }
        }

        stage('Build Docker Image') {
            when {
                expression {
                    env.BUILD_MODE == 'production'
                }
            }

            steps {
                script {

                    env.IMAGE =
                        "${DOCKER_REPO}:${IMAGE_TAG}"

                    sh """
                        docker build \
                          -t ${IMAGE} \
                          .
                    """
                }
            }
        }

    }
}
// new jenkinfiles