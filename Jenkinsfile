pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/app" \
                      -w /app/backend \
                      node:20-alpine \
                      sh -c "npm ci"
                    
                    docker run --rm \
                      -v "$WORKSPACE:/app" \
                      -w /app/frontend \
                      node:20-alpine \
                      sh -c "npm ci && npm run build"
                '''
            }
        }

        stage('SCA - Trivy') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/src" \
                      aquasec/trivy:latest \
                      fs \
                      --scanners vuln \
                      --severity HIGH,CRITICAL \
                      /src
                '''
            }
        }
    }
}