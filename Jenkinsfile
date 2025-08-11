pipeline {
    agent any
    
    environment {
        GITHUB_TOKEN = credentials('github-token')
        DOCKER_IMAGE = 'prometheus:latest'
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
                        # Check if Go is actually available and working
                        GO_AVAILABLE=false
                        if command -v go &> /dev/null; then
                            if go version &> /dev/null; then
                                GO_AVAILABLE=true
                                echo "Go is already available: $(go version)"
                            else
                                echo "Go command exists but doesn't work, will reinstall"
                            fi
                        fi
                        
                        if [ "$GO_AVAILABLE" = false ]; then
                            echo "Installing Go..."
                            # Use a recent Go version with fallback URLs
                            GO_VERSION="1.21.15"
                            GO_ARCH="linux-amd64"
                            
                            # Try multiple download URLs
                            if wget -q "https://go.dev/dl/go${GO_VERSION}.${GO_ARCH}.tar.gz"; then
                                echo "Downloaded from go.dev"
                            elif wget -q "https://golang.org/dl/go${GO_VERSION}.${GO_ARCH}.tar.gz"; then
                                echo "Downloaded from golang.org"
                            else
                                echo "Failed to download Go from primary sources, trying archive"
                                wget -q "https://archive.org/download/golang-archive/go${GO_VERSION}.${GO_ARCH}.tar.gz" || {
                                    echo "Failed to download Go from all sources"
                                    exit 1
                                }
                            fi
                            
                            sudo tar -C /usr/local -xzf "go${GO_VERSION}.${GO_ARCH}.tar.gz"
                            
                            # Clean up downloaded archive
                            rm -f "go${GO_VERSION}.${GO_ARCH}.tar.gz"
                            
                            echo "Go installation completed"
                        fi
                        
                        # Always ensure Go is in PATH and verify it works
                        export PATH=$PATH:/usr/local/go/bin
                        echo "Updated PATH: $PATH"
                        
                        # Verify Go is working
                        if /usr/local/go/bin/go version; then
                            echo "Go verification successful: $(/usr/local/go/bin/go version)"
                        else
                            echo "Go verification failed"
                            exit 1
                        fi
                        
                        # Create Go workspace if it doesn't exist
                        sudo mkdir -p /var/lib/jenkins/go
                        sudo chown jenkins:jenkins /var/lib/jenkins/go
                        
                        # Export environment variables for this shell session
                        export GOROOT=/usr/local/go
                        export GOPATH=/var/lib/jenkins/go
                        export PATH=/usr/local/go/bin:$PATH
                        
                        echo "Go environment setup complete:"
                        echo "GOROOT: $GOROOT"
                        echo "GOPATH: $GOPATH"
                        echo "PATH: $PATH"
                        
                        # Install Docker if not present
                        if ! command -v docker &> /dev/null; then
                            echo "Installing Docker..."
                            curl -fsSL https://get.docker.com -o get-docker.sh
                            sudo sh get-docker.sh
                            sudo usermod -aG docker $USER
                        fi
                        
                        # Start Docker service
                        sudo systemctl start docker
                        sudo systemctl enable docker
                        
                        echo "Environment setup completed successfully"
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
                        docker build -t ${DOCKER_IMAGE} .
                        
                        if [ $? -ne 0 ]; then
                            echo "Failed to build Prometheus image"
                            exit 1
                        fi
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
    }
    
    post {
        always {
            script {
                // Clean up Docker containers
                sh '''
                    echo "Cleaning up Docker containers..."
                    docker ps -aq | xargs -r docker rm -f || true
                    docker images -q ${DOCKER_IMAGE} | xargs -r docker rmi || true
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
