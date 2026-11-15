pipeline {
    agent any

    triggers {
        githubPush()
        pollSCM('H/2 * * * *')
    }

    environment {
        IMAGE_NAME = 'smart-inventory'
        CONTAINER_NAME = 'smart-inventory-app'
        HOST_PORT = '80'
        CONTAINER_PORT = '5000'
        VOLUME_NAME = 'inventory_data'
        DATABASE_CONTAINER_PATH = '/data'
        HEALTH_URL = "http://localhost:${HOST_PORT}/health"
    }

    stages {
        stage('Checkout Source') {
            steps {
                echo "=========================================="
                echo "STAGE 1: CHECKING OUT SOURCE CODE"
                echo "Build Number: ${BUILD_NUMBER}"
                echo "Commit: ${GIT_COMMIT}"
                echo "Branch: ${GIT_BRANCH}"
                echo "=========================================="
                checkout scm
            }
        }

        stage('Environment & Unit Tests') {
            steps {
                echo "=========================================="
                echo "STAGE 2: INSTALLING DEPENDENCIES & RUNNING PYTEST"
                echo "=========================================="
                sh '''
                    set -e
                    echo "[INFO] Setting up Python virtual environment for testing..."
                    python3 -m venv .venv_ci
                    . .venv_ci/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements-dev.txt
                    
                    echo "[INFO] Executing automated test suite..."
                    pytest -v --tb=short
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "=========================================="
                echo "STAGE 3: BUILDING DOCKER IMAGE"
                echo "Image Tag: ${IMAGE_NAME}:${BUILD_NUMBER}"
                echo "=========================================="
                sh '''
                    set -e
                    echo "[INFO] Building Docker container image..."
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest .
                    echo "[SUCCESS] Docker image successfully built."
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                echo "=========================================="
                echo "STAGE 4: DEPLOYING APPLICATION TO DOCKER"
                echo "=========================================="
                sh '''
                    set -e
                    # Record currently running image tag for rollback if available
                    PREV_IMAGE=$(docker inspect --format='{{.Config.Image}}' ${CONTAINER_NAME} 2>/dev/null || echo "")
                    echo "PREV_IMAGE=${PREV_IMAGE}" > /tmp/jenkins_prev_image.env

                    echo "[INFO] Ensuring persistent Docker volume exists..."
                    docker volume create ${VOLUME_NAME}

                    echo "[INFO] Stopping and removing existing application container if running..."
                    docker stop ${CONTAINER_NAME} 2>/dev/null || true
                    docker rm ${CONTAINER_NAME} 2>/dev/null || true

                    echo "[INFO] Launching updated container with persistent volume mount..."
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${HOST_PORT}:${CONTAINER_PORT} \
                        -v ${VOLUME_NAME}:${DATABASE_CONTAINER_PATH} \
                        -e SECRET_KEY="production-secret-${BUILD_NUMBER}" \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    echo "[SUCCESS] New container launched successfully."
                '''
            }
        }

        stage('Health Verification & Rollback') {
            steps {
                echo "=========================================="
                echo "STAGE 5: VERIFYING APPLICATION HEALTH"
                echo "Endpoint: ${HEALTH_URL}"
                echo "=========================================="
                sh '''
                    set -e
                    echo "[INFO] Awaiting application readiness..."
                    MAX_RETRIES=15
                    COUNT=0
                    HEALTHY=0

                    while [ $COUNT -lt $MAX_RETRIES ]; do
                        COUNT=$((COUNT + 1))
                        echo "[ATTEMPT $COUNT/$MAX_RETRIES] Pinging ${HEALTH_URL}..."
                        HTTP_STATUS=$(curl -s -o /tmp/health_response.json -w "%{http_code}" ${HEALTH_URL} || echo "000")
                        
                        if [ "$HTTP_STATUS" = "200" ]; then
                            echo "[SUCCESS] Received HTTP 200 OK from healthcheck!"
                            cat /tmp/health_response.json
                            echo ""
                            HEALTHY=1
                            break
                        fi
                        echo "[WARN] Status: $HTTP_STATUS. Waiting 4 seconds before retry..."
                        sleep 4
                    done

                    if [ $HEALTHY -ne 1 ]; then
                        echo "[ERROR] Health check failed after $MAX_RETRIES attempts!"
                        echo "[INFO] Container logs:"
                        docker logs --tail 50 ${CONTAINER_NAME} || true
                        
                        # Rollback procedure
                        if [ -f /tmp/jenkins_prev_image.env ]; then
                            . /tmp/jenkins_prev_image.env
                            if [ -n "$PREV_IMAGE" ]; then
                                echo "[ROLLBACK] Rolling back to previous working image: $PREV_IMAGE"
                                docker stop ${CONTAINER_NAME} || true
                                docker rm ${CONTAINER_NAME} || true
                                docker run -d \
                                    --name ${CONTAINER_NAME} \
                                    --restart unless-stopped \
                                    -p ${HOST_PORT}:${CONTAINER_PORT} \
                                    -v ${VOLUME_NAME}:${DATABASE_CONTAINER_PATH} \
                                    $PREV_IMAGE
                                echo "[ROLLBACK] Previous container restored."
                            fi
                        fi
                        exit 1
                    fi

                    echo "=========================================="
                    echo "[DEPLOYMENT COMPLETE] Application is healthy and operational!"
                    echo "=========================================="
                '''
            }
        }
    }

    post {
        always {
            sh '''
                # Clean up CI virtual environment and temp files
                rm -rf .venv_ci /tmp/health_response.json /tmp/jenkins_prev_image.env || true
            '''
        }
        success {
            echo "CI/CD Pipeline Build #${BUILD_NUMBER} completed successfully!"
        }
        failure {
            echo "CI/CD Pipeline Build #${BUILD_NUMBER} failed. Please inspect console logs."
        }
    }
}

