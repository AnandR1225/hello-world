pipeline {
    agent any
    options {
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }
    environment {
        GIT_REPO              = "https://github.com/AnandR1225/hello-world.git"
        GIT_CREDENTIALS_ID    = "github1"

    }
    triggers {
        githubPush()
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout Code') {
            steps {
                script {
                    // TAG_NAME is set automatically when the Multibranch job is
                    // triggered by a discovered tag. BRANCH_NAME covers branch builds.
                    def refName = env.TAG_NAME ?: env.BRANCH_NAME ?: 'build/jenkins-testing'
                    def refPath = env.TAG_NAME ? "refs/tags/${refName}" : "*/${refName}"

                    echo "Checking out: ${refName} (${env.TAG_NAME ? 'tag' : 'branch'})"

                    checkout([$class: 'GitSCM',
                        branches: [[name: refPath]],
                        userRemoteConfigs: [[
                            url: env.GIT_REPO,
                            credentialsId: env.GIT_CREDENTIALS_ID
                        ]]
                    ])

                    env.ACTUAL_BRANCH = refName
                }
            }
        }

        stage('Determine Environment') {
            steps {
                script {
                    if (env.TAG_NAME) {
                        env.DEPLOY_ENV = "production"
                        env.TAG_TYPE   = "release"
                    } else {
                        error("Unsupported trigger: no TAG_NAME present. This pipeline only builds on tag pushes (ref: ${env.ACTUAL_BRANCH}).")
                    }

                    echo """
                    Environment Info
                    ----------------------
                    Ref:    ${env.ACTUAL_BRANCH}
                    Deploy: ${env.DEPLOY_ENV}
                    Repo:   ${env.IMAGE_NAME}
                    Mode:   ${env.TAG_TYPE}
                    """
                }
            }
        }

        stage('Generate Docker Tag') {
            steps {
                script {
                    if (!env.TAG_NAME) {
                        error("No TAG_NAME present — this job wasn't triggered by a tag push. Stopping build.")
                    }
                    env.IMAGE_TAG = env.TAG_NAME
                    echo "FINAL Docker Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Docker Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USER" --password-stdin'
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    def imageFull = "${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                    echo "Building Docker image: ${imageFull}"
                    sh """
                        docker build --pull --no-cache -t ${imageFull} .
                        docker push ${imageFull}
                    """
                    sh "docker logout"
                }
            }
        }
    }
}

//testing jenkinsfile