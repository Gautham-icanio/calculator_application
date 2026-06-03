pipeline {
    agent any

    environment {
        IMAGE_NAME     = "calculator-app"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        CONTAINER_NAME = "calculator"
        APP_PORT       = "3000"
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
            }
        }

        // ─── 2. VALIDATE ────────────────────────────────────────────────
        stage('Validate') {
            steps {
                echo '🔍 Validating project files...'
                sh '''
                    echo "--- Files in workspace ---"
                    ls -la
                    echo ""
                    echo "--- Checking Dockerfile ---"
                    test -f Dockerfile && echo "✅ Dockerfile found" || (echo "❌ Dockerfile missing" && exit 1)
                    echo "--- Checking index.html ---"
                    test -f index.html && echo "✅ index.html found" || (echo "❌ index.html missing" && exit 1)
                '''
            }
        }

        // ─── 3. BUILD DOCKER IMAGE ──────────────────────────────────────
        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                sh """
                    docker build \\
                        --no-cache \\
                        -t ${IMAGE_NAME}:${IMAGE_TAG} \\
                        -t ${IMAGE_NAME}:latest \\
                        .
                """
            }
        }

        // ─── 4. TEST (random port — no conflicts) ───────────────────────
        stage('Test') {
            steps {
                echo '🧪 Running container smoke test...'
                sh """
                    # Remove any leftover test container from a previous failed run
                    docker rm -f test-calc-${BUILD_NUMBER} 2>/dev/null || true

                    # -P lets Docker pick a free ephemeral port — never conflicts
                    docker run -d --name test-calc-${BUILD_NUMBER} \\
                        -P \\
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    # Discover which host port was assigned to container port 80
                    TEST_PORT=\$(docker inspect \\
                        --format='{{(index (index .NetworkSettings.Ports "80/tcp") 0).HostPort}}' \\
                        test-calc-${BUILD_NUMBER})
                    echo "Test container listening on host port: \$TEST_PORT"

                    # Retry until HTTP 200 or timeout (15 × 3 s = 45 s max)
                    STATUS=000
                    COUNT=0
                    RETRIES=15
                    until [ "\$STATUS" = "200" ] || [ "\$COUNT" -ge "\$RETRIES" ]; do
                        sleep 3
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:\$TEST_PORT/ 2>/dev/null || echo "000")
                        COUNT=\$((COUNT+1))
                        echo "Attempt \$COUNT/\$RETRIES — HTTP \$STATUS"
                    done

                    # Always remove the test container
                    docker rm -f test-calc-${BUILD_NUMBER} 2>/dev/null || true

                    if [ "\$STATUS" != "200" ]; then
                        echo "❌ Smoke test FAILED — HTTP \$STATUS"
                        exit 1
                    fi
                    echo "✅ Smoke test PASSED"
                """
            }
        }

        // ─── 5. DEPLOY ──────────────────────────────────────────────────
        stage('Deploy') {
            steps {
                echo "🚀 Deploying ${IMAGE_NAME}:${IMAGE_TAG} on port ${APP_PORT}..."
                sh """
                    # Stop and remove old production container if running
                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true

                    # Run the new container
                    docker run -d \\
                        --name ${CONTAINER_NAME} \\
                        --restart unless-stopped \\
                        -p ${APP_PORT}:80 \\
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "✅ Container started"
                    docker ps --filter "name=${CONTAINER_NAME}"
                """
            }
        }

        // ─── 6. HEALTH CHECK ────────────────────────────────────────────
        stage('Health Check') {
            steps {
                echo '❤️  Running post-deploy health check...'
                sh """
                    STATUS=000
                    COUNT=0
                    RETRIES=15
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

        // ─── 7. CLEANUP OLD IMAGES ──────────────────────────────────────
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
                docker rm -f test-calc-${BUILD_NUMBER} 2>/dev/null || true
                docker rm -f ${CONTAINER_NAME}         2>/dev/null || true
            """
        }
        always {
            echo "Build #${BUILD_NUMBER} finished — Status: ${currentBuild.currentResult}"
        }
    }
}