pipeline {
    agent any
    
    environment {
        GITHUB_TOKEN = credentials('github-token')
        DOCKER_IMAGE = 'prometheus:latest'
        DOCKER_REGISTRY = 'localhost:5000'
        TEST_DIR = 'prometheus-integration-tests'
        GO_VERSION = '1.21.15'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    // Since we're running from within the prometheus repo, 
                    // we can use a simpler checkout approach
                    if (env.CHANGE_ID) {
                        // This is a PR build
                        def repoUrl = env.GIT_URL ?: 'https://github.com/prometheus/prometheus.git'
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: '${GITHUB_REF}']],
                            doGenerateSubmoduleConfigurations: false,
                            extensions: [
                                [$class: 'SubmoduleOption', disableSubmodules: false, recursiveSubmodules: true, trackingSubmodules: false],
                                [$class: 'CleanBeforeCheckout'],
                                [$class: 'CleanCheckout']
                            ],
                            submoduleCfg: [],
                            userRemoteConfigs: [[
                                url: repoUrl,
                                refspec: "+refs/pull/${CHANGE_ID}/head:refs/remotes/origin/pr/${CHANGE_ID}"
                            ]]
                        ])
                    } else {
                        // This is a branch build, use standard checkout
                        checkout scm
                    }
                }
            }
        }
        
        stage('Setup Environment') {
            steps {
                script {
                    sh '''
                        echo "Setting up environment..."
                        
                        # Install Go if not present
                        if ! command -v go >/dev/null 2>&1; then
                            echo "Installing Go ${GO_VERSION}..."
                            
                            # Clean up any existing broken installations
                            sudo rm -rf /usr/local/go || true
                            
                            # Download and install Go
                            cd /tmp
                            wget -q "https://golang.org/dl/go${GO_VERSION}.linux-amd64.tar.gz"
                            
                            if [ -f "go${GO_VERSION}.linux-amd64.tar.gz" ]; then
                                sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"
                                sudo ln -sf /usr/local/go/bin/go /usr/bin/go
                                rm -f "go${GO_VERSION}.linux-amd64.tar.gz"
                                echo "Go installation completed"
                            else
                                echo "ERROR: Failed to download Go"
                                exit 1
                            fi
                        else
                            echo "Go is already available: $(go version)"
                        fi
                        
                        # Set up Go environment
                        export PATH=/usr/local/go/bin:$PATH
                        export GOROOT=/usr/local/go
                        export GOPATH=/var/lib/jenkins/go
                        
                        # Create Go workspace
                        sudo mkdir -p /var/lib/jenkins/go
                        sudo chown jenkins:jenkins /var/lib/jenkins/go
                        
                        # Verify Go installation
                        if ! go version; then
                            echo "ERROR: Go verification failed"
                            exit 1
                        fi
                        
                        # Install Docker if not present
                        if ! command -v docker >/dev/null 2>&1; then
                            echo "Installing Docker..."
                            curl -fsSL https://get-docker.com -o get-docker.sh
                            sudo sh get-docker.sh
                            sudo usermod -aG docker jenkins
                        fi
                        
                        # Start Docker service
                        sudo systemctl start docker || true
                        sudo systemctl enable docker || true
                        
                        # Ensure Jenkins user has Docker access
                        sudo usermod -aG docker jenkins || true
                        sudo chmod 666 /var/run/docker.sock || true
                        
                        echo "Environment setup completed"
                    '''
                }
            }
        }
        
        stage('Build Prometheus Binaries') {
            steps {
                script {
                    sh '''
                        echo "Building Prometheus binaries..."
                        
                        # Ensure we're in the right directory
                        cd /var/lib/jenkins/workspace/${JOB_NAME}
                        
                        # Set up Go environment again (for this stage)
                        export PATH=/usr/local/go/bin:$PATH
                        export GOROOT=/usr/local/go
                        export GOPATH=/var/lib/jenkins/go
                        
                        # Download Go dependencies
                        echo "Downloading Go dependencies..."
                        go mod download
                        
                        # Build the Go binaries
                        echo "Building Prometheus and Promtool..."
                        make build
                        
                        # Verify binaries were created
                        echo "Verifying build outputs..."
                        if [ ! -f ".build/linux-amd64/prometheus" ]; then
                            echo "ERROR: Prometheus binary not found after build"
                            ls -la .build/linux-amd64/ || echo "Build directory not found"
                            exit 1
                        fi
                        
                        if [ ! -f ".build/linux-amd64/promtool" ]; then
                            echo "ERROR: Promtool binary not found after build"
                            ls -la .build/linux-amd64/ || echo "Build directory not found"
                            exit 1
                        fi
                        
                        echo "Binary build completed successfully"
                        echo "Build outputs:"
                        ls -la .build/linux-amd64/
                        echo "Binary sizes:"
                        du -h .build/linux-amd64/prometheus .build/linux-amd64/promtool
                    '''
                }
            }
        }
        
        stage('Test Basic Commands') {
            steps {
                script {
                    sh '''
                        echo "Testing basic commands..."
                        echo "Go version: $(go version)"
                        echo "Docker version: $(docker --version)"
                        echo "Basic commands test completed"
                    '''
                }
            }
        }
        
        stage('Setup Docker Registry') {
            steps {
                script {
                    sh '''
                        echo "Setting up local Docker registry..."
                        
                        # Check if registry container is already running
                        if docker ps | grep -q "registry:2"; then
                            echo "Docker registry is already running"
                        else
                            echo "Starting Docker registry..."
                            # Stop any existing registry containers
                            docker ps -q --filter "name=registry" | xargs -r docker stop
                            docker ps -aq --filter "name=registry" | xargs -r docker rm
                            
                            # Start a new registry container
                            docker run -d \
                                --name registry \
                                --restart=unless-stopped \
                                -p 5000:5000 \
                                -v /var/lib/jenkins/registry:/var/lib/registry \
                                registry:2
                            
                            # Wait for registry to be ready
                            echo "Waiting for registry to be ready..."
                            sleep 10
                            
                            # Test registry connectivity
                            if curl -s http://localhost:5000/v2/ > /dev/null; then
                                echo "Docker registry is ready"
                            else
                                echo "Warning: Registry may not be fully ready yet"
                            fi
                        fi
                        
                        # Configure Docker to allow insecure registry
                        if ! grep -q "insecure-registry" /etc/docker/daemon.json 2>/dev/null; then
                            echo "Configuring Docker daemon for insecure registry..."
                            sudo mkdir -p /etc/docker
                            echo '{"insecure-registries": ["localhost:5000"]}' | sudo tee /etc/docker/daemon.json
                            sudo systemctl restart docker
                            sleep 5
                        fi
                        
                        echo "Docker registry setup completed"
                    '''
                }
            }
        }
        
        stage('Build Prometheus Docker Image') {
            steps {
                script {
                    sh '''
                        # Build Prometheus Docker image using pre-built binaries
                        echo "Building Prometheus Docker image..."
                        
                        # Verify binaries exist before Docker build
                        if [ ! -f ".build/linux-amd64/prometheus" ] || [ ! -f ".build/linux-amd64/promtool" ]; then
                            echo "ERROR: Required binaries not found. Did the previous build stage complete successfully?"
                            exit 1
                        fi
                        
                        # Build with local registry tag
                        LOCAL_IMAGE="${DOCKER_REGISTRY}/${DOCKER_IMAGE}"
                        docker build -t ${DOCKER_IMAGE} -t ${LOCAL_IMAGE} .
                        
                        if [ $? -ne 0 ]; then
                            echo "Failed to build Prometheus image"
                            exit 1
                        fi
                        
                        # Push to local registry
                        echo "Pushing image to local registry..."
                        docker push ${LOCAL_IMAGE}
                        
                        if [ $? -ne 0 ]; then
                            echo "Failed to push to local registry"
                            exit 1
                        fi
                        
                        echo "Successfully built and pushed: ${LOCAL_IMAGE}"
                    '''
                }
            }
        }
        
        stage('Clone Integration Tests') {
            steps {
                script {
                    sh '''
                        # Clone the integration tests repository
                        echo "Cloning prometheus-integration-tests..."
                        if [ -d "${TEST_DIR}" ]; then
                            rm -rf "${TEST_DIR}"
                        fi
                        git clone https://github.com/Project-AI-Aurora/prometheus-integration-tests.git "${TEST_DIR}"
                        cd "${TEST_DIR}"
                        go mod download
                    '''
                }
            }
        }
        
        stage('Run Integration Tests') {
            steps {
                script {
                    dir(TEST_DIR) {
                        sh '''
                            # Run the integration tests
                            echo "Running Prometheus integration tests..."
                            go test -v -timeout 10m ./...
                            
                            # Capture test results
                            TEST_EXIT_CODE=$?
                            
                            # Generate test report
                            echo "Generating test report..."
                            go test -v -json ./... > test-results.json 2>&1 || true
                            
                            exit $TEST_EXIT_CODE
                        '''
                    }
                }
            }
            post {
                always {
                    script {
                        // Archive test results
                        archiveArtifacts artifacts: "${TEST_DIR}/test-results.json,${TEST_DIR}/logs/*", allowEmptyArchive: true
                        
                        // Publish test results to GitHub
                        publishTestResults()
                    }
                }
            }
        }
        
        stage('Generate Coverage Report') {
            steps {
                script {
                    dir(TEST_DIR) {
                        sh '''
                            # Generate coverage report
                            echo "Generating coverage report..."
                            go test -coverprofile=coverage.out ./...
                            go tool cover -html=coverage.out -o coverage.html
                        '''
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${TEST_DIR}/coverage.out,${TEST_DIR}/coverage.html", allowEmptyArchive: true
                }
            }
        }
        
        // This stage only runs when PUSH_TO_DOCKERHUB=true
        // To enable: Add dockerhub-username and dockerhub-password credentials to Jenkins
        stage('Push to Docker Hub (Optional)') {
            when {
                environment name: 'PUSH_TO_DOCKERHUB', value: 'true'
            }
            steps {
                script {
                    sh '''
                        echo "Checking Docker Hub credentials..."
                        
                        # Check if Docker Hub credentials are available
                        if [ -z "$DOCKERHUB_USERNAME" ] || [ -z "$DOCKERHUB_PASSWORD" ]; then
                            echo "Docker Hub credentials not available, skipping push"
                            echo "To enable Docker Hub push, add credentials to Jenkins and set PUSH_TO_DOCKERHUB=true"
                            exit 0
                        fi
                        
                        echo "Pushing image to Docker Hub..."
                        
                        # Login to Docker Hub
                        echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_USERNAME --password-stdin
                        
                        # Tag for Docker Hub
                        DOCKERHUB_IMAGE="${DOCKERHUB_USERNAME}/prometheus:${BUILD_NUMBER}"
                        docker tag ${DOCKER_IMAGE} ${DOCKERHUB_IMAGE}
                        
                        # Push to Docker Hub
                        docker push ${DOCKERHUB_IMAGE}
                        
                        if [ $? -eq 0 ]; then
                            echo "Successfully pushed to Docker Hub: ${DOCKERHUB_IMAGE}"
                        else
                            echo "Failed to push to Docker Hub"
                            exit 1
                        fi
                    '''
                }
            }
        }
        
        // Show status when Docker Hub is not enabled
        stage('Docker Hub Status') {
            when {
                not { environment name: 'PUSH_TO_DOCKERHUB', value: 'true' }
            }
            steps {
                script {
                    echo "Docker Hub push is disabled"
                    echo "To enable: Set PUSH_TO_DOCKERHUB=true and add Docker Hub credentials to Jenkins"
                    echo "Current setup: Using local Docker registry at ${DOCKER_REGISTRY}"
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Clean up Docker containers and images
                sh '''
                    echo "Cleaning up Docker containers and images..."
                    docker ps -aq | xargs -r docker rm -f || true
                    
                    # Clean up local images
                    docker images -q ${DOCKER_IMAGE} | xargs -r docker rmi || true
                    docker images -q ${DOCKER_REGISTRY}/${DOCKER_IMAGE} | xargs -r docker rmi || true
                    
                    # Keep registry running but clean up old images
                    echo "Registry cleanup completed"
                '''
                
                // Post results to GitHub PR if this is a PR build
                if (env.CHANGE_ID) {
                    postResultsToGitHub()
                }
            }
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}

def publishTestResults() {
    script {
        def testResults = readJSON file: "${TEST_DIR}/test-results.json"
        def summary = generateTestSummary(testResults)
        
        // Create a detailed comment for the PR
        def comment = """
## 🧪 Integration Test Results

### Summary
- **Status**: ${currentBuild.result == 'SUCCESS' ? '✅ PASSED' : '❌ FAILED'}
- **Build**: [#${env.BUILD_NUMBER}](${env.BUILD_URL})
- **Duration**: ${currentBuild.durationString}

### Test Results
${summary}

### Coverage
- Coverage report: [View Coverage](${env.BUILD_URL}artifact/${TEST_DIR}/coverage.html)

### Logs
- [Test Logs](${env.BUILD_URL}artifact/${TEST_DIR}/logs/)
- [Build Logs](${env.BUILD_URL}console)

---
*This comment was automatically generated by Jenkins*
        """
        
        // Store comment for later posting
        env.PR_COMMENT = comment
    }
}

def generateTestSummary(testResults) {
    def passed = 0
    def failed = 0
    def summary = ""
    
    testResults.each { result ->
        if (result.Action == "pass") {
            passed++
            summary += "- ✅ ${result.Test}\n"
        } else if (result.Action == "fail") {
            failed++
            summary += "- ❌ ${result.Test}: ${result.Output ?: 'Failed'}\n"
        }
    }
    
    return """
**Tests Passed**: ${passed}
**Tests Failed**: ${failed}
**Total Tests**: ${passed + failed}

**Details**:
${summary}
    """
}

def postResultsToGitHub() {
    script {
        if (env.PR_COMMENT) {
            def prNumber = env.CHANGE_ID
            // Calculate repository name from GIT_URL
            def repo = env.GIT_URL ? env.GIT_URL.replaceAll('.*github\\.com[:/]([^/]+/[^/]+?)(\\.git)?$', '$1') : 'prometheus/prometheus'
            
            // Post comment to GitHub PR
            sh """
                curl -X POST \
                    -H "Authorization: token ${GITHUB_TOKEN}" \
                    -H "Accept: application/vnd.github.v3+json" \
                    https://api.github.com/repos/${repo}/issues/${prNumber}/comments \
                    -d '{"body": ${env.PR_COMMENT}}'
            """
        }
    }
}
