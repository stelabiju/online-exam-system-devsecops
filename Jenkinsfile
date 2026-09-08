pipeline {

    agent any

    environment {
        BACKEND_IMAGE  = "stelabiju/online-exam-backend:${BUILD_NUMBER}"
        FRONTEND_IMAGE = "stelabiju/online-exam-frontend:${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    echo "Building backend..."
                    docker run --rm \
                      -v "$WORKSPACE:/app" \
                      -w /app/backend \
                      node:20-alpine \
                      sh -c "npm ci"

                    echo "Building frontend..."
                    docker run --rm \
                      -v "$WORKSPACE:/app" \
                      -w /app/frontend \
                      node:20-alpine \
                      sh -c "npm ci && npm run build"

                    echo "Build completed successfully."
                '''
            }
        }

        stage('Static Code Analysis') {
            steps {
                withCredentials([
                    string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')
                ]) {
                    sh '''
                        docker run --rm \
                          -v "$WORKSPACE:/usr/src" \
                          sonarsource/sonar-scanner-cli:latest \
                          sonar-scanner \
                          -Dsonar.projectKey=online-exam-system \
                          -Dsonar.sources=/usr/src/backend,/usr/src/frontend \
                          -Dsonar.host.url=http://54.91.185.154:9000 \
                          -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        stage('SCA - Trivy') {
            steps {
                sh '''
                    echo "Running Trivy filesystem vulnerability scan..."

                    docker run --rm \
                      -v "$WORKSPACE:/src" \
                      aquasec/trivy:latest \
                      fs \
                      --scanners vuln \
                      --severity HIGH,CRITICAL \
                      --format json \
                      --output /src/trivy-sca-report.json \
                      /src
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    echo "Building backend image..."
                    docker build \
                      -t ${BACKEND_IMAGE} \
                      ./backend

                    echo "Building frontend image..."
                    docker build \
                      -t ${FRONTEND_IMAGE} \
                      ./frontend

                    echo "Docker images built successfully."
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    echo "Scanning backend image..."

                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      aquasec/trivy:latest \
                      image \
                      --severity HIGH,CRITICAL \
                      ${BACKEND_IMAGE}

                    echo "Scanning frontend image..."

                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      aquasec/trivy:latest \
                      image \
                      --severity HIGH,CRITICAL \
                      ${FRONTEND_IMAGE}
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                script {
                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        'docker-cred'
                    ) {
                        docker.image("${BACKEND_IMAGE}").push()
                        docker.image("${FRONTEND_IMAGE}").push()
                    }
                }
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                sh '''
                    echo "Updating Kubernetes image tags..."

                    sed -i "s/IMAGE_TAG/${BUILD_NUMBER}/g" \
                      k8s/backend-deployment.yaml

                    sed -i "s/IMAGE_TAG/${BUILD_NUMBER}/g" \
                      k8s/frontend-deployment.yaml

                    echo "Updated manifests:"
                    grep "image:" k8s/backend-deployment.yaml
                    grep "image:" k8s/frontend-deployment.yaml
                '''
            }
        }

        stage('Commit and Push Changes') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'Github_jenkins',
                        usernameVariable: 'stelabiju',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        git config user.email "jenkins@localhost"
                        git config user.name "Jenkins"

                        git add k8s/backend-deployment.yaml
                        git add k8s/frontend-deployment.yaml

                        git commit \
                          -m "Update Kubernetes images to build ${BUILD_NUMBER}" \
                          || echo "No manifest changes to commit"

                        git push https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/stelabiju/online-exam-system-devsecops.git HEAD:main
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-sca-report.json',
                             allowEmptyArchive: true
        }

        success {
            echo 'CI/CD pipeline completed successfully.'
            echo 'Argo CD will detect the updated Kubernetes manifests.'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage and security scan results.'
        }
    }
}
