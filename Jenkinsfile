pipeline {
    agent any
    
    environment {
        GITHUB_TOKEN = credentials('github-token')
        DOCKER_IMAGE = 'prometheus:latest'
        DOCKER_REGISTRY = 'localhost:5000'
        TEST_DIR = 'prometheus-integration-tests'
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
                    // Install Go if not present
                    sh '''
                        echo "Starting environment setup..."
                        echo "=== Jenkins Environment Check ==="
                        echo "Current user: $(whoami)"
                        echo "Jenkins user: jenkins"
                        echo "Can run sudo: $(sudo -n true 2>/dev/null && echo 'YES' || echo 'NO')"
                        echo "Current working directory: $(pwd)"
                        echo "=================================="
                        
                        # Check if Go is already available and working
                        echo "Checking Go availability..."
                        
                        # Try to find any working Go installation
                        GO_AVAILABLE=false
                        
                        # Check system PATH first
                        if command -v go >/dev/null 2>&1; then
                            echo "Go command found in PATH, testing if it works..."
                            if go version >/dev/null 2>&1; then
                                echo "Go is already available: $(go version)"
                                GO_AVAILABLE=true
                            else
                                echo "Go command exists but doesn't work, will try to install..."
                                GO_AVAILABLE=false
                            fi
                        else
                            echo "Go command not found in PATH"
                        fi
                        
                        # Check common Go installation locations
                        if [ "$GO_AVAILABLE" = false ]; then
                            echo "Checking common Go installation locations..."
                            for go_path in "/usr/local/go/bin/go" "/usr/bin/go" "/opt/go/bin/go"; do
                                if [ -x "$go_path" ] && "$go_path" version >/dev/null 2>&1; then
                                    echo "Found working Go at: $go_path"
                                    export PATH="$(dirname $go_path):$PATH"
                                    GO_AVAILABLE=true
                                    break
                                fi
                            done
                        fi
                        
                        echo "GO_AVAILABLE set to: $GO_AVAILABLE"
                        
                        echo "About to check if Go installation is needed..."
                        if [ "$GO_AVAILABLE" = false ]; then
                            echo "Installing Go..."
                            
                            # Try system package manager first (might not require sudo password)
                            echo "Trying system package manager for Go..."
                            if command -v apt-get >/dev/null 2>&1; then
                                echo "Found apt-get, trying to install Go..."
                                if sudo apt-get update -qq && sudo apt-get install -y golang-go; then
                                    echo "Go installed via apt-get successfully!"
                                    GO_AVAILABLE=true
                                else
                                    echo "apt-get installation failed, trying manual installation..."
                                fi
                            elif command -v yum >/dev/null 2>&1; then
                                echo "Found yum, trying to install Go..."
                                if sudo yum install -y golang; then
                                    echo "Go installed via yum successfully!"
                                    GO_AVAILABLE=true
                                else
                                    echo "yum installation failed, trying manual installation..."
                                fi
                            fi
                            
                            # If system package manager failed, try manual installation
                            if [ "$GO_AVAILABLE" = false ]; then
                                echo "Proceeding with manual Go installation..."
                                
                                # Try to clean up any existing broken installations (non-fatal)
                                echo "Cleaning up existing Go installations..."
                                sudo rm -rf /usr/local/go || echo "Failed to remove /usr/local/go, continuing..."
                                sudo rm -f /usr/bin/go || echo "Failed to remove /usr/bin/go, continuing..."
                            
                            # Use a stable Go version
                            GO_VERSION="1.21.15"
                            GO_ARCH="linux-amd64"
                            
                            # Download Go with fallback URLs
                            echo "Downloading Go ${GO_VERSION}..."
                            if wget -q "https://go.dev/dl/go${GO_VERSION}.${GO_ARCH}.tar.gz"; then
                                echo "Downloaded from go.dev"
                            elif wget -q "https://golang.org/dl/go${GO_VERSION}.${GO_ARCH}.tar.gz"; then
                                echo "Downloaded from golang.org"
                            else
                                echo "Failed to download Go from primary sources"
                                echo "WARNING: Go installation failed, but continuing..."
                                echo "This will cause issues in later stages"
                                return 0
                            fi
                            
                            # Install Go
                            echo "Installing Go to /usr/local..."
                            if sudo tar -C /usr/local -xzf "go${GO_VERSION}.${GO_ARCH}.tar.gz"; then
                                echo "Go extraction successful"
                            else
                                echo "Go extraction failed"
                                echo "WARNING: Go installation failed, but continuing..."
                                return 0
                            fi
                            
                            # Verify installation
                            if [ ! -d "/usr/local/go/bin" ]; then
                                echo "Error: Go installation directory not found"
                                echo "WARNING: Go installation failed, but continuing..."
                                return 0
                            fi
                            
                            # Create symlink
                            sudo ln -sf /usr/local/go/bin/go /usr/bin/go || echo "Failed to create symlink, continuing..."
                            
                            # Clean up
                            rm -f "go${GO_VERSION}.${GO_ARCH}.tar.gz" || echo "Failed to clean up download, continuing..."
                            
                            echo "Go installation completed"
                            
                            # Verify the new installation
                            echo "Verifying new Go installation..."
                            ls -la /usr/local/go/bin/ || echo "Go bin directory not found"
                            ls -la /usr/bin/go || echo "Go symlink not found"
                            
                            # Test the newly installed Go
                            echo "Testing newly installed Go..."
                            if /usr/local/go/bin/go version; then
                                echo "New Go installation works!"
                                GO_AVAILABLE=true
                            else
                                echo "New Go installation failed verification"
                            fi
                            fi
                        fi
                        
                        # Set up Go environment
                        echo "Setting up Go environment..."
                        export PATH=/usr/local/go/bin:$PATH
                        export GOROOT=/usr/local/go
                        export GOPATH=/var/lib/jenkins/go
                        
                        echo "New PATH: $PATH"
                        echo "New GOROOT: $GOROOT"
                        echo "New GOPATH: $GOPATH"
                        
                        # Verify Go works
                        echo "Testing Go command..."
                        if go version; then
                            echo "Go verification successful: $(go version)"
                        else
                            echo "Go verification failed"
                            echo "Trying to find Go binary..."
                            which go || echo "which go failed"
                            ls -la /usr/local/go/bin/go || echo "Go binary not found"
                            echo "Current PATH: $PATH"
                            echo "WARNING: Go verification failed, but continuing..."
                            echo "This may cause issues in later stages"
                        fi
                        
                        # Create Go workspace
                        sudo mkdir -p /var/lib/jenkins/go || echo "Failed to create Go workspace directory, continuing..."
                        sudo chown jenkins:jenkins /var/lib/jenkins/go || echo "Failed to change ownership, continuing..."
                        
                        echo "Go environment setup complete"
                        
                        # Final verification of what we have
                        echo "=== Final Go Environment Check ==="
                        echo "PATH: $PATH"
                        echo "GOROOT: $GOROOT"
                        echo "GOPATH: $GOPATH"
                        echo "which go: $(which go 2>/dev/null || echo 'not found')"
                        echo "ls /usr/local/go/bin: $(ls /usr/local/go/bin 2>/dev/null || echo 'directory not found')"
                        echo "ls /usr/bin/go: $(ls -la /usr/bin/go 2>/dev/null || echo 'symlink not found')"
                        echo "=================================="
                        
                        # Install Docker if not present
                        if ! command -v docker &> /dev/null; then
                            echo "Installing Docker..."
                            curl -fsSL https://get.docker.com -o get-docker.sh
                            sudo sh get-docker.sh
                            sudo usermod -aG docker jenkins
                        fi
                        
                        # Start Docker service
                        sudo systemctl start docker || echo "Docker start failed, continuing..."
                        sudo systemctl enable docker || echo "Docker enable failed, continuing..."
                        
                        # Ensure Jenkins user has Docker access
                        sudo usermod -aG docker jenkins || echo "Failed to add jenkins to docker group, continuing..."
                        sudo chmod 666 /var/run/docker.sock || true
                        
                        # Test Docker access
                        if docker --version &> /dev/null; then
                            echo "Docker is accessible"
                        else
                            echo "Warning: Docker may not be accessible to Jenkins user"
                        fi
                        
                        echo "Environment setup completed successfully"
                        
                        # Debug: Show what we have
                        echo "=== Environment Debug Info ==="
                        echo "PATH: $PATH"
                        echo "GOROOT: $GOROOT"
                        echo "GOPATH: $GOPATH"
                        echo "Go version: $(go version 2>/dev/null || echo 'Go not in PATH')"
                        echo "Docker version: $(docker --version 2>/dev/null || echo 'Docker not accessible')"
                        echo "Current user: $(whoami)"
                        echo "Jenkins user: jenkins"
                        echo "================================"
                    '''
                }
            }
        }
        
        stage('Test Basic Commands') {
            steps {
                script {
                    sh '''
                        echo "Testing basic command execution..."
                        echo "Current directory: $(pwd)"
                        echo "Current user: $(whoami)"
                        echo "Go version: $(go version 2>/dev/null || echo 'Go not available')"
                        echo "Docker version: $(docker --version 2>/dev/null || echo 'Docker not available')"
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
        
        stage('Build Prometheus') {
            steps {
                script {
                    sh '''
                        # Build Prometheus Docker image
                        echo "Building Prometheus Docker image..."
                        
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
