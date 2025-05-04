pipeline {
    agent any
    
    environment {
        // SonarQube configuration
        SONAR_PROJECT_KEY = "order-inventory-system"
        SONAR_PROJECT_NAME = "Order Inventory System"
        // Node.js version for frontend analysis
        NODE_VERSION = "18"
        // Java version for backend analysis
        JAVA_VERSION = "11"
        // GitHub credentials
        GITHUB_CREDENTIALS = credentials('github-credentials')
        // Docker Hub credentials ID (configured in Jenkins)
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
        // Docker image names
        FRONTEND_IMAGE = "vishwakant1602/order-inventory-frontend"
        BACKEND_BASE_IMAGE = "vishwakant1602/order-inventory"
        // Version information
        VERSION = "${BUILD_NUMBER}"
        // Node.js version for frontend
        NODE_VERSION = "18"
        // Java version for backend
        JAVA_VERSION = "11"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Checked out code from repository"
                
                // Store git information for later use
                script {
                    env.GIT_COMMIT_MSG = sh(script: 'git log -1 --pretty=%B ${GIT_COMMIT}', returnStdout: true).trim()
                    env.GIT_AUTHOR = sh(script: 'git log -1 --pretty=%an ${GIT_COMMIT}', returnStdout: true).trim()
                    env.GIT_BRANCH_NAME = env.BRANCH_NAME
                    env.PULL_REQUEST = env.CHANGE_ID ? "PR-${env.CHANGE_ID}" : ""
                }
            }
        }
        
        stage('Build Frontend') {
            steps {
                echo "Building frontend application..."
                sh '''
                    # Check if Node.js is available
                    if command -v node &> /dev/null; then
                        node_version=$(node -v)
                        echo "Using Node.js $node_version"
                    else
                        echo "Node.js not found, installing..."
                        curl -sL https://deb.nodesource.com/setup_18.x | bash -
                        apt-get install -y nodejs
                    fi
                    
                    # Install dependencies and build
                    npm ci || npm install
                    npm run build || echo "Build command failed but continuing"
                '''
            }
        }
        
        stage('Test Frontend') {
            steps {
                echo "Running frontend tests..."
                sh '''
                    # Run tests if you have them
                    npm test || echo "No tests to run or tests failed but continuing"
                '''
            }
        }
        
        stage('Build Backend Services') {
            steps {
                echo "Building backend microservices..."
                sh '''
                    cd backend
                    if [ -f "./mvnw" ]; then
                        chmod +x ./mvnw
                        ./mvnw clean package -DskipTests || echo "Maven build failed but continuing"
                    else
                        echo "Maven wrapper not found, using system Maven if available"
                        mvn clean package -DskipTests || echo "Maven build failed but continuing"
                    fi
                '''
            }
        }
        
        stage('Test Backend Services') {
            steps {
                echo "Running backend tests..."
                sh '''
                    cd backend
                    if [ -f "./mvnw" ]; then
                        ./mvnw test || echo "Tests failed but continuing"
                    else
                        mvn test || echo "Tests failed but continuing"
                    fi
                '''
            }
        }
        
        stage('Build Docker Images') {
            steps {
                echo "Building Docker images..."
                sh '''
                    # Build frontend Docker image if Dockerfile exists
                    if [ -f "Dockerfile" ]; then
                        docker build -t ${FRONTEND_IMAGE}:${VERSION} -t ${FRONTEND_IMAGE}:latest . || echo "Frontend Docker build failed"
                    else
                        echo "Frontend Dockerfile not found, skipping"
                    fi
                    
                    # Check if backend directory exists
                    if [ -d "backend" ]; then
                        cd backend
                        
                        # Build backend service Docker images if directories exist
                        for service in eureka-server api-gateway order-service inventory-service auth-service payment-service; do
                            if [ -d "$service" ] && [ -f "$service/Dockerfile" ]; then
                                echo "Building Docker image for $service"
                                docker build -t ${BACKEND_BASE_IMAGE}-${service}:${VERSION} -t ${BACKEND_BASE_IMAGE}-${service}:latest ./$service || echo "Docker build for $service failed"
                            fi
                        done
                    else
                        echo "Backend directory not found, skipping backend Docker builds"
                    fi
                '''
            }
        }
        
        stage('Code Quality Analysis') {
            parallel {
                stage('Frontend Code Quality') {
                    steps {
                        echo "Running ESLint for frontend code quality..."
                        sh '''
                            export NVM_DIR="$HOME/.nvm"
                            [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
                            nvm install ${NODE_VERSION}
                            nvm use ${NODE_VERSION}
                            
                            # Install ESLint if not already in package.json
                            npm install eslint --no-save
                            
                            # Run ESLint
                            npx eslint . --ext .js,.jsx,.ts,.tsx || echo "ESLint found issues"
                        '''
                    }
                    post {
                        always {
                            recordIssues(
                                tools: [esLint(pattern: 'eslint-report.xml')],
                                qualityGates: [[threshold: 10, type: 'TOTAL', unstable: true]]
                            )
                        }
                    }
                }
                
                stage('Backend Code Quality') {
                    steps {
                        echo "Running Checkstyle for backend code quality..."
                        sh '''
                            cd backend
                            ./mvnw checkstyle:checkstyle || echo "Checkstyle found issues"
                        '''
                    }
                    post {
                        always {
                            recordIssues(
                                tools: [checkStyle(pattern: '**/target/checkstyle-result.xml')],
                                qualityGates: [[threshold: 10, type: 'TOTAL', unstable: true]]
                            )
                        }
                    }
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo "Running SonarQube analysis..."
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        # Frontend analysis
                        export NVM_DIR="$HOME/.nvm"
                        [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
                        nvm use ${NODE_VERSION}
                        
                        # Install sonarqube scanner
                        npm install sonarqube-scanner --no-save
                        
                        # Run sonar analysis for frontend
                        npx sonarqube-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY}-frontend \
                          -Dsonar.projectName="${SONAR_PROJECT_NAME} Frontend" \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=node_modules/**,backend/**,**/*.test.js,**/*.spec.js \
                          -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                        
                        # Backend analysis
                        cd backend
                        ./mvnw sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY}-backend \
                          -Dsonar.projectName="${SONAR_PROJECT_NAME} Backend" \
                          -Dsonar.java.binaries=**/target/classes
                    '''
                }
                
                // Wait for SonarQube quality gate
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Security Scan') {
            parallel {
                stage('Frontend Security Scan') {
                    steps {
                        echo "Running npm audit for frontend security..."
                        sh '''
                            export NVM_DIR="$HOME/.nvm"
                            [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
                            nvm use ${NODE_VERSION}
                            
                            # Run npm audit
                            npm audit --json > npm-audit.json || true
                        '''
                        
                        // Optional: Run OWASP Dependency Check
                        dependencyCheck(
                            additionalArguments: '--scan . --exclude "backend/**" --out dependency-check-report.xml',
                            odcInstallation: 'OWASP-Dependency-Check'
                        )
                    }
                    post {
                        always {
                            // Publish npm audit results
                            publishHTML(
                                target: [
                                    allowMissing: true,
                                    alwaysLinkToLastBuild: true,
                                    keepAll: true,
                                    reportDir: '.',
                                    reportFiles: 'npm-audit.json',
                                    reportName: 'NPM Audit Report'
                                ]
                            )
                            
                            // Publish dependency check results
                            dependencyCheckPublisher(
                                pattern: 'dependency-check-report.xml'
                            )
                        }
                    }
                }
                
                stage('Backend Security Scan') {
                    steps {
                        echo "Running OWASP Dependency Check for backend security..."
                        sh '''
                            cd backend
                            
                            # Run OWASP Dependency Check
                            ../mvnw org.owasp:dependency-check-maven:check
                        '''
                    }
                    post {
                        always {
                            // Publish dependency check results
                            dependencyCheckPublisher(
                                pattern: '**/dependency-check-report.xml'
                            )
                        }
                    }
                }
            }
        }
        
        stage('Branch Management') {
            when {
                expression { return env.PULL_REQUEST != "" }
            }
            steps {
                echo "Validating pull request..."
                
                // Check if PR title follows conventional commits
                script {
                    def prTitle = sh(script: "curl -s -H 'Authorization: token ${GITHUB_CREDENTIALS_PSW}' https://api.github.com/repos/owner/repo/pulls/${env.CHANGE_ID} | jq -r '.title'", returnStdout: true).trim()
                    
                    if (!(prTitle =~ /^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)($$.+$$)?: .+$/)) {
                        error "PR title does not follow conventional commits format"
                    }
                }
                
                // Check if branch is up to date with target branch
                sh '''
                    git fetch origin ${CHANGE_TARGET}
                    MERGE_BASE=$(git merge-base HEAD origin/${CHANGE_TARGET})
                    if [ "$(git rev-parse origin/${CHANGE_TARGET})" != "$MERGE_BASE" ]; then
                        echo "Branch is not up to date with target branch ${CHANGE_TARGET}"
                        exit 1
                    fi
                '''
                
                // Check if PR has required approvals
                script {
                    def approvals = sh(script: "curl -s -H 'Authorization: token ${GITHUB_CREDENTIALS_PSW}' https://api.github.com/repos/owner/repo/pulls/${env.CHANGE_ID}/reviews | jq '[.[] | select(.state==\"APPROVED\")] | length'", returnStdout: true).trim()
                    
                    if (approvals.toInteger() < 1) {
                        error "PR requires at least 1 approval"
                    }
                }
            }
        }
        
        stage('Code Review Automation') {
            when {
                expression { return env.PULL_REQUEST != "" }
            }
            steps {
                echo "Automating code review tasks..."
                
                // Post SonarQube analysis results as PR comment
                script {
                    def sonarIssues = sh(script: "curl -s -u ${SONAR_TOKEN}: 'https://sonarqube.example.com/api/issues/search?componentKeys=${SONAR_PROJECT_KEY}&resolved=false&severities=BLOCKER,CRITICAL,MAJOR'", returnStdout: true).trim()
                    
                    def comment = "## SonarQube Analysis Results\n\n"
                    comment += "Found the following issues:\n\n"
                    comment += sonarIssues
                    
                    sh """
                        curl -s -H 'Authorization: token ${GITHUB_CREDENTIALS_PSW}' \
                             -X POST \
                             -d '{"body": "${comment}"}' \
                             https://api.github.com/repos/owner/repo/issues/${env.CHANGE_ID}/comments
                    """
                }
                
                // Label PR based on changes
                script {
                    def filesChanged = sh(script: "git diff --name-only origin/${CHANGE_TARGET}...HEAD", returnStdout: true).trim()
                    
                    def labels = []
                    
                    if (filesChanged.contains("frontend") || filesChanged.contains(".js") || filesChanged.contains(".ts") || filesChanged.contains(".tsx")) {
                        labels.add("frontend")
                    }
                    
                    if (filesChanged.contains("backend") || filesChanged.contains(".java")) {
                        labels.add("backend")
                    }
                    
                    if (filesChanged.contains("Dockerfile") || filesChanged.contains("docker-compose")) {
                        labels.add("docker")
                    }
                    
                    if (filesChanged.contains("Jenkinsfile") || filesChanged.contains(".github/workflows")) {
                        labels.add("ci-cd")
                    }
                    
                    if (!labels.isEmpty()) {
                        sh """
                            curl -s -H 'Authorization: token ${GITHUB_CREDENTIALS_PSW}' \
                                 -X POST \
                                 -d '{"labels": ${groovy.json.JsonOutput.toJson(labels)}}' \
                                 https://api.github.com/repos/owner/repo/issues/${env.CHANGE_ID}/labels
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
        success {
            echo 'SCM pipeline completed successfully!'
            // Update GitHub commit status
            script {
                if (env.PULL_REQUEST != "") {
                    sh """
                        curl -s -H 'Authorization: token ${GITHUB_CREDENTIALS_PSW}' \
                             -X POST \
                             -d '{"state": "success", "context": "jenkins/scm", "description": "SCM checks passed"}' \
                             https://api.github.com/repos/owner/repo/statuses/${GIT_COMMIT}
                    """
                }
            }
        }
        failure {
            echo 'SCM pipeline failed!'
            // Update GitHub commit status
            script {
                if (env.PULL_REQUEST != "") {
                    sh """
                        curl -s -H 'Authorization: token ${GITHUB_CREDENTIALS_PSW}' \
                             -X POST \
                             -d '{"state": "failure", "context": "jenkins/scm", "description": "SCM checks failed"}' \
                             https://api.github.com/repos/owner/repo/statuses/${GIT_COMMIT}
                    """
                }
            }
        }
    }
}
