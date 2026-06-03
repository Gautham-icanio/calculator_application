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

        // ─── 4. TEST ────────────────────────────────────────────────────
        // NOTE: Jenkins runs inside Docker, so we put the test container on
        // a shared network and curl it by container name — no port mapping needed.
        stage('Test') {
            steps {
                echo '🧪 Running container smoke test...'
                sh """
                    # Clean up any leftover test container
                    docker rm -f test-calc-${BUILD_NUMBER} 2>/dev/null || true

                    # Create a dedicated test network
                    docker network create test-net-${BUILD_NUMBER} 2>/dev/null || true

                    # Start the test container on that network (no port binding needed)
                    docker run -d \\
                        --name test-calc-${BUILD_NUMBER} \\
                        --network test-net-${BUILD_NUMBER} \\
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    # Also connect the Jenkins container to the same network
                    JENKINS_CONTAINER=\$(hostname)
                    docker network connect test-net-${BUILD_NUMBER} \$JENKINS_CONTAINER 2>/dev/null || true

                    # Retry curl by container name (not localhost)
                    STATUS="000"
                    COUNT=0
                    RETRIES=15
                    until [ "\$STATUS" = "200" ] || [ "\$COUNT" -ge "\$RETRIES" ]; do
                        sleep 3
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://test-calc-${BUILD_NUMBER}:80/ 2>/dev/null)
                        COUNT=\$((COUNT+1))
                        echo "Attempt \$COUNT/\$RETRIES — HTTP \$STATUS"
                    done

                    # Disconnect Jenkins from the test network
                    docker network disconnect test-net-${BUILD_NUMBER} \$JENKINS_CONTAINER 2>/dev/null || true

                    # Clean up test container and network
                    docker rm -f test-calc-${BUILD_NUMBER} 2>/dev/null || true
                    docker network rm test-net-${BUILD_NUMBER} 2>/dev/null || true

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
                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true

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
        // Same fix: Jenkins is inside Docker, so connect to the app container's network.
        stage('Health Check') {
            steps {
                echo '❤️  Running post-deploy health check...'
                sh """
                    # Connect Jenkins container to the calculator app's network
                    JENKINS_CONTAINER=\$(hostname)
                    docker network connect bridge \$JENKINS_CONTAINER 2>/dev/null || true

                    STATUS="000"
                    COUNT=0
                    RETRIES=15
                    until [ "\$STATUS" = "200" ] || [ "\$COUNT" -ge "\$RETRIES" ]; do
                        sleep 3
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://${CONTAINER_NAME}:80/ 2>/dev/null)
                        COUNT=\$((COUNT+1))
                        echo "Attempt \$COUNT/\$RETRIES — HTTP \$STATUS"
                    done

                    if [ "\$STATUS" != "200" ]; then
                        echo "❌ Health check FAILED"
                        docker logs ${CONTAINER_NAME}
                        exit 1
                    fi
                    echo "✅ App is healthy!"
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
                docker network rm test-net-${BUILD_NUMBER} 2>/dev/null || true
                docker rm -f ${CONTAINER_NAME} 2>/dev/null || true
            """
        }
        always {
            echo "Build #${BUILD_NUMBER} finished — Status: ${currentBuild.currentResult}"
        }
    }
}