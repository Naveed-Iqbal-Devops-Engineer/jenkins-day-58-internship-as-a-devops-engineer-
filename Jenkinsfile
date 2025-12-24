pipeline {
    agent any

    environment {
        GITHUB_CREDENTIALS_ID     = 'github56'
        DOCKERHUB_CREDENTIALS_ID  = 'docker56'
        BRANCH_NAME               = "${env.BRANCH_NAME}"
        IMAGE_TAG                 = "${BRANCH_NAME}-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('GitHub Auth (Optional)') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDENTIALS_ID}",
                    usernameVariable: 'GITHUB_USER',
                    passwordVariable: 'GITHUB_PASS')]) {
                    echo "🔐 GitHub credentials loaded for user: ${env.GITHUB_USER}"
                }
            }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_PASS')]) {
                    sh '''
                        echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin
                    '''
                }
            }
        }

        stage('Build & Tag Images') {
            steps {
                script {
                    // ✅ Use env.DOCKERHUB_USER to avoid MissingPropertyException
                    env.FRONTEND_TAG_DH = "${env.DOCKERHUB_USER}/jenkins-day65:frontend-${IMAGE_TAG}"
                    env.BACKEND_TAG_DH  = "${env.DOCKERHUB_USER}/jenkins-day65:backend-${IMAGE_TAG}"

                    sh """
                        docker build -t ${BACKEND_TAG_DH} ./backend
                        docker build -t ${FRONTEND_TAG_DH} ./frontend
                    """
                }
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh """
                    docker push ${BACKEND_TAG_DH}
                    docker push ${FRONTEND_TAG_DH}
                """
            }
        }

        stage('Prepare .env for Compose') {
            steps {
                script {
                    writeFile file: '.env', text: """BACKEND_IMAGE=${BACKEND_TAG_DH}
FRONTEND_IMAGE=${FRONTEND_TAG_DH}
"""
                }
            }
        }

        stage('Approval for Staging / Prod Deploy') {
            when {
                anyOf {
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                input message: "Deploy to ${BRANCH_NAME} environment?", ok: "Yes, Deploy"
            }
        }

        stage('Deploy Environment') {
            steps {
                sh """
                    docker-compose --env-file .env down
                    docker-compose --env-file .env pull
                    docker-compose --env-file .env up -d --remove-orphans
                """
            }
        }

        stage('Cleanup Local Images') {
            steps {
                sh """
                    docker rmi ${BACKEND_TAG_DH} ${FRONTEND_TAG_DH} || true
                """
            }
        }
    }

    post {
        success {
            echo "✅ ${BRANCH_NAME} environment deployed successfully using Docker Hub repo: jenkins-day65!"
        }
        failure {
            echo "❌ Deployment failed for ${BRANCH_NAME}. Check logs."
        }
    }
}
