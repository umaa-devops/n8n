pipeline {
    agent any

    environment {
        REGISTRY = 'umaatturu/n8n'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKERFILE_PATH = 'docker/images/n8n/Dockerfile'
        REGISTRY_CREDENTIALS = 'dockerhub-credentials'
        PNPM_VERSION = '10.18.3'
        GIT_URL = 'https://github.com/umaa-devops/n8n.git'
        GIT_BRANCH = 'umaa/jenkins-pipeline'
        TEMP_BUILD_DIR = "${WORKSPACE}/n8n_build_temp"
    }

    stages {
        stage('Prepare Temporary Build Directory') {
            steps {
                sh '''
                    set -ex
                    mkdir -p "$TEMP_BUILD_DIR"
                    ls -la "$TEMP_BUILD_DIR"
                '''
            }
        }

        stage('Build n8n inside Node Container') {
            steps {
                sh '''
                    set -ex

                    CONTAINER_ID=$(docker create -w /app node:22-bullseye bash -c "
                        git clone -b ${GIT_BRANCH} ${GIT_URL} /app
                        cd /app
                        corepack enable
                        corepack prepare pnpm@${PNPM_VERSION} --activate
                        pnpm install --frozen-lockfile
                        pnpm build:n8n
                    ")

                    docker start -ai $CONTAINER_ID

                    docker cp $CONTAINER_ID:/app/compiled "$TEMP_BUILD_DIR"

                    docker rm -f $CONTAINER_ID || true

                    if [ ! -d "$TEMP_BUILD_DIR/compiled" ]; then
                        echo "❌ compiled directory missing"
                        exit 1
                    fi

                    ls -la "$TEMP_BUILD_DIR/compiled"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -ex

                    cp -r "$TEMP_BUILD_DIR/compiled" ./compiled

                    docker build -t ${REGISTRY}:${IMAGE_TAG} -f ${DOCKERFILE_PATH} .
                    docker tag ${REGISTRY}:${IMAGE_TAG} ${REGISTRY}:latest
                '''
            }
        }

        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: REGISTRY_CREDENTIALS,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${REGISTRY}:${IMAGE_TAG}
                        docker push ${REGISTRY}:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                echo "Cleaning up local Docker images and temp build files..."
                docker rmi ${REGISTRY}:${IMAGE_TAG} || true
                docker rmi ${REGISTRY}:latest || true
                rm -rf "$TEMP_BUILD_DIR"
            '''
        }
        success {
            echo "✅ Build & push successful: ${BUILD_URL}"
        }
        failure {
            echo "❌ Build failed: ${BUILD_URL}"
        }
    }
}