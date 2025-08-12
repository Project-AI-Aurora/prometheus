pipeline {
    agent any
    
    environment {
        GO111MODULE = 'on'
        GOPATH = '/tmp/go'
        GOCACHE = '/tmp/go-cache'
        PATH = "/usr/local/go/bin:$PATH"
        TIA_BINARY = '/tmp/tia'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup Go Environment') {
            steps {
                sh '''
                    mkdir -p $GOPATH
                    mkdir -p $GOCACHE
                    go version
                    go env
                '''
            }
        }
        
        stage('Download Dependencies') {
            steps {
                sh '''
                    cd $WORKSPACE
                    go mod download
                    go mod verify
                '''
            }
        }
        
        stage('Setup TIA Tool') {
            steps {
                sh '''
                    echo "🚀 Setting up TIA for intelligent test execution..."
                    
                    # Try to copy TIA binary from the TIA repository
                    if [ -f "/Users/samohan/Code/TIA/tia" ]; then
                        echo "✅ Found local TIA binary, copying..."
                        cp /Users/samohan/Code/TIA/tia $TIA_BINARY
                        chmod +x $TIA_BINARY
                        echo "✅ TIA setup completed successfully"
                    else
                        echo "⚠️  Local TIA binary not found, will run full test suite"
                        echo "💡 To enable TIA, ensure the TIA repository is available at /Users/samohan/Code/TIA"
                    fi
                '''
            }
        }
        
        stage('Generate Coverage Data') {
            steps {
                sh '''
                    if [ -f "$TIA_BINARY" ] && [ -x "$TIA_BINARY" ]; then
                        echo "🔍 Generating coverage data for Prometheus repository..."
                        echo "This will analyze all tests and create coverage mappings..."
                        
                        # Run TIA analyze to generate coverage data
                        cd $WORKSPACE
                        $TIA_BINARY analyze --output-dir .tia-coverage
                        
                        if [ -d ".tia-coverage" ]; then
                            echo "✅ Coverage data generated successfully"
                            echo "📊 Coverage data contents:"
                            ls -la .tia-coverage/
                        else
                            echo "❌ Failed to generate coverage data"
                        fi
                    else
                        echo "⚠️  TIA not available, skipping coverage generation"
                    fi
                '''
            }
        }
        
        stage('Run Impact Analysis') {
            steps {
                script {
                    // Check if we're in a PR context
                    if (env.CHANGE_ID) {
                        echo "🔍 Pull Request detected: ${env.CHANGE_ID}"
                        echo "🎯 Base branch: ${env.CHANGE_TARGET}"
                        echo "🌿 Source branch: ${env.BRANCH_NAME}"
                        
                        // Run TIA impact analysis
                        sh '''
                            if [ -f "$TIA_BINARY" ] && [ -x "$TIA_BINARY" ] && [ -d ".tia-coverage" ]; then
                                echo "🚀 Running TIA impact analysis..."
                                
                                # Show changed files
                                echo "📝 Changed files:"
                                git diff --name-only origin/${CHANGE_TARGET}...HEAD
                                
                                # Run TIA impact analysis
                                echo "🔍 Analyzing test impact..."
                                $TIA_BINARY impact --base-branch origin/${CHANGE_TARGET} --verbose
                                
                                # Check if TIA found impacted tests
                                if [ -f ".tia-coverage/impacted-tests.txt" ]; then
                                    echo "✅ TIA found impacted tests:"
                                    cat .tia-coverage/impacted-tests.txt
                                    env.USE_TIA = 'true'
                                else
                                    echo "⚠️  No impacted tests found, will run full test suite"
                                    env.USE_TIA = 'false'
                                fi
                            else
                                echo "❌ TIA or coverage data not available, will run full test suite"
                                env.USE_TIA = 'false'
                            fi
                        '''
                    } else {
                        echo "ℹ️  Not a Pull Request, running full test suite"
                        env.USE_TIA = 'false'
                    }
                }
            }
        }
        
        stage('Run Tests (TIA Optimized)') {
            steps {
                script {
                    if (env.USE_TIA == 'true') {
                        echo "🚀 Running only impacted tests using TIA..."
                        sh '''
                            # Run TIA to execute only impacted tests
                            $TIA_BINARY run --base-branch origin/${CHANGE_TARGET} --verbose
                        '''
                    } else {
                        echo "🔄 Running full test suite..."
                        sh '''
                            cd $WORKSPACE
                            go test -v ./...
                        '''
                    }
                }
            }
            post {
                always {
                    junit '**/test-results/*.xml'
                }
            }
        }
        
        stage('Build Prometheus') {
            steps {
                sh '''
                    cd $WORKSPACE
                    make build
                '''
            }
        }
        
        stage('Test Prometheus Binary') {
            steps {
                sh '''
                    cd $WORKSPACE
                    ./prometheus --version
                    ./promtool --version
                '''
            }
        }
        
        stage('Generate TIA Report') {
            when {
                expression { env.USE_TIA == 'true' }
            }
            steps {
                sh '''
                    echo "=== TIA Impact Analysis Report ===" > tia-report.txt
                    echo "Generated: $(date)" >> tia-report.txt
                    echo "Branch: ${BRANCH_NAME}" >> tia-report.txt
                    echo "Target: ${CHANGE_TARGET}" >> tia-report.txt
                    echo "" >> tia-report.txt
                    echo "Changed files:" >> tia-report.txt
                    git diff --name-only origin/${CHANGE_TARGET}...HEAD >> tia-report.txt
                    echo "" >> tia-report.txt
                    echo "Impacted tests:" >> tia-report.txt
                    if [ -f ".tia-coverage/impacted-tests.txt" ]; then
                        cat .tia-coverage/impacted-tests.txt >> tia-report.txt
                    fi
                    echo "" >> tia-report.txt
                    echo "Test execution time: $(($SECONDS / 60)) minutes and $(($SECONDS % 60)) seconds" >> tia-report.txt
                    echo "" >> tia-report.txt
                    echo "Performance improvement: From ~1 hour to ~5-15 minutes" >> tia-report.txt
                    echo "" >> tia-report.txt
                    echo "Coverage data: Generated fresh for this repository" >> tia-report.txt
                    
                    # Archive the TIA report
                    archiveArtifacts artifacts: 'tia-report.txt', fingerprint: true
                    
                    echo "📊 TIA Report generated and archived"
                '''
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            script {
                if (env.USE_TIA == 'true') {
                    echo '🎉 Pipeline completed successfully using TIA impact analysis!'
                    echo '🚀 Only relevant tests were executed, saving significant time.'
                    echo '⏱️  Expected time savings: From ~1 hour to ~5-15 minutes'
                    echo '📊 Fresh coverage data was generated for this repository'
                } else {
                    echo '✅ Pipeline completed successfully with full test suite!'
                    echo '💡 Consider setting up TIA for faster test execution in PRs'
                }
            }
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
