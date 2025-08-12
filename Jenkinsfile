pipeline {
    agent any
    
    environment {
        GO111MODULE = 'on'
        GOPATH = '/tmp/go'
        GOCACHE = '/tmp/go-cache'
        PATH = "/usr/local/go/bin:$PATH"
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
        
        stage('Run Unit Tests') {
            steps {
                sh '''
                    cd $WORKSPACE
                    go test -v ./...
                '''
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
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
