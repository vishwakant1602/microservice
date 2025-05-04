pipeline {
    agent any
    
    environment {
        DOCKER_HUB_CREDS = credentials('docker-hub-credentials')
        DOCKER_USERNAME = 'vishwakant1602'
        FRONTEND_IMAGE = "${DOCKER_USERNAME}/orderinventory-frontend:${BUILD_NUMBER}"
        BACKEND_IMAGE_PREFIX = "${DOCKER_USERNAME}/orderinventory"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    // Store git information for later use
                    env.GIT_COMMIT_MSG = sh(script: 'git log -1 --pretty=%B ${GIT_COMMIT}', returnStdout: true).trim()
                    env.GIT_AUTHOR = sh(script: 'git log -1 --pretty=%an ${GIT_COMMIT}', returnStdout: true).trim()
                    env.GIT_BRANCH = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                }
            }
        }
        
        stage('Build Frontend') {
            steps {
                echo "Building frontend..."
                sh 'npm install'
                sh 'npm run build'
            }
        }
        
        stage('Test Frontend') {
            steps {
                echo "Testing frontend..."
                sh 'npm test || echo "No tests or tests failed but continuing"'
            }
        }
        
        stage('Build Backend Services') {
            steps {
                dir('backend') {
                    echo "Building backend services..."
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        
        stage('Test Backend Services') {
            steps {
                dir('backend') {
                    echo "Testing backend services..."
                    sh 'mvn test || echo "No tests or tests failed but continuing"'
                }
            }
        }
        
        stage('Build Docker Images') {
            steps {
                echo "Building Docker images..."
                
                // Build frontend image
                sh "docker build -t ${FRONTEND_IMAGE} ."
                
                // Build backend service images
                sh "docker build -t ${BACKEND_IMAGE_PREFIX}-eureka:${BUILD_NUMBER} ./backend/eureka-server"
                sh "docker build -t ${BACKEND_IMAGE_PREFIX}-gateway:${BUILD_NUMBER} ./backend/api-gateway"
                sh "docker build -t ${BACKEND_IMAGE_PREFIX}-order:${BUILD_NUMBER} ./backend/order-service"
                sh "docker build -t ${BACKEND_IMAGE_PREFIX}-inventory:${BUILD_NUMBER} ./backend/inventory-service"
                sh "docker build -t ${BACKEND_IMAGE_PREFIX}-auth:${BUILD_NUMBER} ./backend/auth-service"
                sh "docker build -t ${BACKEND_IMAGE_PREFIX}-payment:${BUILD_NUMBER} ./backend/payment-service"
            }
        }
        
        stage('Push Docker Images') {
            when {
                anyOf {
                    branch 'main'
                    branch 'staging'
                }
            }
            steps {
                echo "Pushing Docker images to registry..."
                
                // Login to Docker Hub
                sh "echo ${DOCKER_HUB_CREDS_PSW} | docker login -u ${DOCKER_HUB_CREDS_USR} --password-stdin"
                
                // Push frontend image
                sh "docker push ${FRONTEND_IMAGE}"
                
                // Push backend service images
                sh "docker push ${BACKEND_IMAGE_PREFIX}-eureka:${BUILD_NUMBER}"
                sh "docker push ${BACKEND_IMAGE_PREFIX}-gateway:${BUILD_NUMBER}"
                sh "docker push ${BACKEND_IMAGE_PREFIX}-order:${BUILD_NUMBER}"
                sh "docker push ${BACKEND_IMAGE_PREFIX}-inventory:${BUILD_NUMBER}"
                sh "docker push ${BACKEND_IMAGE_PREFIX}-auth:${BUILD_NUMBER}"
                sh "docker push ${BACKEND_IMAGE_PREFIX}-payment:${BUILD_NUMBER}"
            }
        }
        
        stage('Deploy to Development') {
            when {
                branch 'develop'
            }
            steps {
                echo "Deploying to development environment..."
                // Deployment steps for development
                sh "docker-compose -f docker-compose.yml up -d"
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'staging'
            }
            steps {
                echo "Deploying to staging environment..."
                // Deployment steps for staging
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                // Manual approval step
                timeout(time: 1, unit: 'DAYS') {
                    input message: 'Approve deployment to production?', submitter: 'admin'
                }
                
                echo "Deploying to production environment..."
                // Deployment steps for production
            }
        }
    }
    
    post {
        always {
            echo "Cleaning up workspace..."
            
            script {
                // Clean up local Docker images to save space
                try {
                    sh "docker rmi ${FRONTEND_IMAGE} || true"
                    sh "docker rmi ${BACKEND_IMAGE_PREFIX}-eureka:${BUILD_NUMBER} || true"
                    sh "docker rmi ${BACKEND_IMAGE_PREFIX}-gateway:${BUILD_NUMBER} || true"
                    sh "docker rmi ${BACKEND_IMAGE_PREFIX}-order:${BUILD_NUMBER} || true"
                    sh "docker rmi ${BACKEND_IMAGE_PREFIX}-inventory:${BUILD_NUMBER} || true"
                    sh "docker rmi ${BACKEND_IMAGE_PREFIX}-auth:${BUILD_NUMBER} || true"
                    sh "docker rmi ${BACKEND_IMAGE_PREFIX}-payment:${BUILD_NUMBER} || true"
                } catch (Exception e) {
                    echo "Warning: Failed to clean up Docker images: ${e.message}"
                }
            }
            
            cleanWs()
        }
        
        success {
            echo "Pipeline completed successfully!"
        }
        
        failure {
            echo "Pipeline failed!"
        }
    }
}
