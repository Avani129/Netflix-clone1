pipeline {
    agent any

    environment {
        SCANNER_HOME             = tool('sonar-scanner')
        GIT_REPO                 = "https://github.com/Anandreddy125/Netflix-clone.git"
        GIT_CREDENTIALS_ID       = "terra-github"
        DOCKER_CREDENTIALS_ID    = "anand-dockerhub"
    }

    parameters {
        // For non-multibranch jobs: select branch manually
        choice(name: 'BRANCH_NAME', choices: ['main', 'master'], description: 'Select branch to build')
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                script {
                    // Determine branch name safely
                    def branchName = env.BRANCH_NAME ?: params.BRANCH_NAME
                    echo "🔹 Checking out branch: ${branchName}"

                    checkout([$class: 'GitSCM',
                        branches: [[name: "*/${branchName}"]],
                        userRemoteConfigs: [[
                            url: "${GIT_REPO}",
                            credentialsId: "${GIT_CREDENTIALS_ID}"
                        ]]
                    ])

                    env.ACTUAL_BRANCH = branchName
                }
            }
        }

        stage('Determine Environment') {
            steps {
                script {
                    if (env.ACTUAL_BRANCH == "dev") {
                        env.DEPLOY_ENV                = "staging"
                        env.IMAGE_NAME                = "anrs125/farhan-testing"
                        env.KUBERNETES_CREDENTIALS_ID = "reports-staging"
                        env.DEPLOY_FILE               = "jenkins/deploy.yaml"
                        env.TAG_TYPE                  = "commit"
                        env.SONAR_PROJECT             = "Staging-reports"
                    } else if (env.ACTUAL_BRANCH == "main") {
                        env.DEPLOY_ENV                = "production"
                        env.IMAGE_NAME                = "anrs125/sample-private"
                        env.KUBERNETES_CREDENTIALS_ID = "reports-prod"
                        env.DEPLOY_FILE               = "kubernetes/service.yaml"
                        env.TAG_TYPE                  = "release"
                        env.SONAR_PROJECT             = "testing-k3s"
                    } else {
                        error("❌ Unsupported branch: ${env.ACTUAL_BRANCH}")
                    }

                    echo """
                    🌿 Environment Details:
                    -----------------------
                    Environment: ${env.DEPLOY_ENV}
                    Docker Image: ${env.IMAGE_NAME}
                    Kubernetes Creds: ${env.KUBERNETES_CREDENTIALS_ID}
                    Deploy File: ${env.DEPLOY_FILE}
                    Sonar Project: ${env.SONAR_PROJECT}
                    """
                }
            }
        }

        stage('Set Build Tag') {
            steps {
                script {
                    if (env.TAG_TYPE == "commit") {
                        def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                        env.IMAGE_TAG = "${DEPLOY_ENV}-${commitId}"
                    } else {
                        def tagName = sh(script: "git describe --tags --exact-match HEAD 2>/dev/null || true", returnStdout: true).trim()
                        if (!tagName) {
                            error("❌ No release tag found. Production builds require a Git tag.")
                        }
                        env.IMAGE_TAG = tagName
                    }
                    echo "🏷️ Docker Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT} \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=${env.SONAR_HOST_URL} \
                            -Dsonar.login=${env.SONAR_AUTH_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh """
                            echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USER} --password-stdin
                            docker build -t ${IMAGE_NAME}:${IMAGE_TAG} --no-cache .
                            docker push ${IMAGE_NAME}:${IMAGE_TAG}
                            docker logout
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "🚀 Deploying ${IMAGE_NAME}:${IMAGE_TAG} to ${DEPLOY_ENV} using ${DEPLOY_FILE}"
                    withKubeConfig(credentialsId: KUBERNETES_CREDENTIALS_ID) {
                        sh """
                            sed -i 's|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g' ${DEPLOY_FILE}
                            kubectl apply -f ${DEPLOY_FILE}
                            kubectl rollout status deployment/anrs -n reports
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up workspace after build..."
            cleanWs()
        }
        success {
            script {
                echo "✅ Deployment completed successfully for ${env.DEPLOY_ENV ?: 'unknown'}"
            }
        }
        failure {
            script {
                echo "❌ Deployment failed for ${env.DEPLOY_ENV ?: 'unknown'}"
            }
        }
    }
}
