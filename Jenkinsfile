pipeline {
    agent any

    environment {
        IMAGE_NAME     = "calculator-app"
        CONTAINER_NAME = "calculator"
        APP_PORT       = "4652"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Test') {
            steps {
                sh """
                    docker rm -f test-calc 2>/dev/null || true
                    docker network create test-net 2>/dev/null || true
                    docker run -d --name test-calc --network test-net ${IMAGE_NAME}:${BUILD_NUMBER}
                    docker network connect test-net \$(hostname)
                    sleep 3
                    STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://test-calc:80/)
                    docker network disconnect test-net \$(hostname)
                    docker rm -f test-calc
                    docker network rm test-net
                    [ "\$STATUS" = "200" ] && echo "✅ Test PASSED" || (echo "❌ Test FAILED: HTTP \$STATUS" && exit 1)
                """
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true
                    docker run -d --name ${CONTAINER_NAME} --restart unless-stopped -p ${APP_PORT}:80 ${IMAGE_NAME}:${BUILD_NUMBER}
                    echo "✅ Deployed at http://localhost:${APP_PORT}"
                """
            }
        }
    }

    post {
        failure {
            sh """
                docker rm -f test-calc 2>/dev/null || true
                docker network rm test-net 2>/dev/null || true
            """
        }
        always {
            echo "Build #${BUILD_NUMBER} — ${currentBuild.currentResult}"
        }
    }
}