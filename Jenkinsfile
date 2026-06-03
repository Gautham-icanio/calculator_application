pipeline {
    agent any

    environment {
        IMAGE_NAME    = "calculator-app"
        IMAGE_TAG     = "${BUILD_NUMBER}"
        CONTAINER_NAME = "calculator"
        APP_PORT      = "3000"
        // If pushing to Docker Hub, set these in Jenkins credentials:
        // DOCKER_HUB_REPO = "yourdockerhubuser/calculator-app"
        // DOCKER_CREDENTIALS_ID = "dockerhub-creds"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    stages {

        // ─── 1. CHECKOUT ────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo '📥 Cloning repository...'
                checkout scm
                // If using a specific repo:
                // git branch: 'main', url: 'https://github.com/youruser/calculator-app.git'
            }
        }

        // ─── 2. LINT / VALIDATE ─────────────────────────────────────────
        stage('Validate') {
            steps {
                echo '🔍 Validating project files...'
                sh '''
                    echo "--- Files in workspace ---"
                    ls -la
                    echo ""
                    echo "--- Checking Dockerfile exists ---"
                    test -f Dockerfile && echo "✅ Dockerfile found" || (echo "❌ Dockerfile missing" && exit 1)
                    echo ""
                    echo "--- Checking index.html exists ---"
                    test -f index.html && echo "✅ index.html found" || (echo "❌ index.html missing" && exit 1)
                '''
            }
        }

        // ─── 3. BUILD DOCKER IMAGE ──────────────────────────────────────
        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                sh """
                    docker build \
                        --no-cache \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} \
                        -t ${IMAGE_NAME}:latest \
                        .
                """
            }
        }

        // ─── 4. TEST ────────────────────────────────────────────────────
        stage('Test') {
            steps {
                echo '🧪 Running container smoke test...'
                sh """
                    # Start a temporary container
                    docker run -d --name test-calc-${BUILD_NUMBER} \
                        -p 3001:80 \
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    # Wait for nginx to be ready (retry up to 15 times)
                    STATUS=000
                    RETRIES=15
                    COUNT=0
                    until [ "\$STATUS" = "200" ] || [ "\$COUNT" -ge "\$RETRIES" ]; do
                        sleep 3
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3001/ 2>/dev/null || echo "000")
                        COUNT=\$((COUNT+1))
                        echo "Attempt \$COUNT/\$RETRIES — HTTP \$STATUS"
                    done

                    # Clean up test container
                    docker stop test-calc-${BUILD_NUMBER} && docker rm test-calc-${BUILD_NUMBER}

                    # Assert status code
                    if [ "\$STATUS" != "200" ]; then
                        echo "❌ Smoke test FAILED — HTTP \$STATUS"
                        exit 1
                    fi
                    echo "✅ Smoke test PASSED"
                """
            }
        }

        // ─── 5. PUSH TO REGISTRY (Optional) ────────────────────────────
        // Uncomment this stage if you want to push to Docker Hub
        /*
        stage('Push to Docker Hub') {
            steps {
                echo '📤 Pushing image to Docker Hub...'
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_HUB_REPO}:${IMAGE_TAG}
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_HUB_REPO}:latest
                        docker push ${DOCKER_HUB_REPO}:${IMAGE_TAG}
                        docker push ${DOCKER_HUB_REPO}:latest
                        docker logout
                    """
                }
            }
        }
        */

        // ─── 6. DEPLOY ──────────────────────────────────────────────────
        stage('Deploy') {
            steps {
                echo "🚀 Deploying ${IMAGE_NAME}:${IMAGE_TAG} on port ${APP_PORT}..."
                sh """
                    # Stop and remove old container if running
                    docker stop ${CONTAINER_NAME} 2>/dev/null || true
                    docker rm   ${CONTAINER_NAME} 2>/dev/null || true

                    # Run the new container
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${APP_PORT}:80 \
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "✅ Container started"
                    docker ps --filter "name=${CONTAINER_NAME}"
                """
            }
        }

        // ─── 7. HEALTH CHECK ────────────────────────────────────────────
        stage('Health Check') {
            steps {
                echo '❤️  Running post-deploy health check...'
                sh """
                    STATUS=000
                    RETRIES=15
                    COUNT=0
                    until [ "\$STATUS" = "200" ] || [ "\$COUNT" -ge "\$RETRIES" ]; do
                        sleep 3
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${APP_PORT}/ 2>/dev/null || echo "000")
                        COUNT=\$((COUNT+1))
                        echo "Attempt \$COUNT/\$RETRIES — HTTP \$STATUS"
                    done
                    if [ "\$STATUS" != "200" ]; then
                        echo "❌ Health check FAILED"
                        docker logs ${CONTAINER_NAME}
                        exit 1
                    fi
                    echo "✅ App is healthy at http://localhost:${APP_PORT}"
                """
            }
        }

        // ─── 8. CLEANUP OLD IMAGES ──────────────────────────────────────
        stage('Cleanup') {
            steps {
                echo '🧹 Removing dangling images...'
                sh "docker image prune -f"
            }
        }
    }

    // ─── POST ACTIONS ───────────────────────────────────────────────────
    post {
        success {
            echo """
            ╔══════════════════════════════════╗
            ║  ✅  BUILD & DEPLOY SUCCESSFUL   ║
            ║  App: http://localhost:${APP_PORT}  ║
            ╚══════════════════════════════════╝
            """
        }
        failure {
            echo '❌ Pipeline FAILED — check logs above'
            sh """
                docker stop ${CONTAINER_NAME}  2>/dev/null || true
                docker rm   ${CONTAINER_NAME}  2>/dev/null || true
            """
        }
        always {
            echo "Build #${BUILD_NUMBER} finished — Status: ${currentBuild.currentResult}"
        }
    }
}