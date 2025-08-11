# Docker Setup Guide for Jenkins Pipeline

## Overview

This Jenkins pipeline now supports multiple Docker image management strategies:

1. **Local Docker Registry** (Default & Recommended)
2. **Docker Hub** (Optional)
3. **Local Images Only** (Fallback)

## 1. Local Docker Registry (Recommended)

### What It Does
- Sets up a local Docker registry at `localhost:5000`
- Stores images persistently in `/var/lib/jenkins/registry`
- Faster builds and no external dependencies
- Better security (images stay within your network)

### How It Works
```bash
# Registry runs on port 5000
docker run -d --name registry -p 5000:5000 -v /var/lib/jenkins/registry:/var/lib/registry registry:2

# Images are tagged and pushed locally
LOCAL_IMAGE="localhost:5000/prometheus:latest"
docker build -t prometheus:latest -t ${LOCAL_IMAGE} .
docker push ${LOCAL_IMAGE}
```

### Benefits
- ✅ **Fast**: No network uploads
- ✅ **Secure**: Images stay local
- ✅ **Reliable**: No external service dependencies
- ✅ **Versioned**: Build numbers and tags preserved

## 2. Docker Hub (Optional)

### When to Use
- Sharing images with external teams
- Deploying to cloud environments
- Public distribution

### Setup Required
1. **Jenkins Credentials**:
   - `dockerhub-username`: Your Docker Hub username
   - `dockerhub-password`: Your Docker Hub password/token

2. **Environment Variable**:
   ```bash
   PUSH_TO_DOCKERHUB=true
   ```

### How It Works
```bash
# Only runs when PUSH_TO_DOCKERHUB=true
docker login -u $DOCKERHUB_USERNAME --password-stdin
docker tag prometheus:latest ${DOCKERHUB_USERNAME}/prometheus:${BUILD_NUMBER}
docker push ${DOCKERHUB_IMAGE}
```

## 3. Local Images Only (Fallback)

### What It Does
- Builds images locally without pushing anywhere
- Good for testing and development
- No registry setup required

### How It Works
```bash
docker build -t prometheus:latest .
# Image stays local, no push
```

## Pipeline Flow

```
Setup Environment → Setup Docker Registry → Build Prometheus → [Optional: Push to Docker Hub]
```

## Configuration Options

### Environment Variables
```groovy
environment {
    DOCKER_IMAGE = 'prometheus:latest'           // Image name
    DOCKER_REGISTRY = 'localhost:5000'           // Local registry URL
    DOCKERHUB_USERNAME = credentials('dockerhub-username')  // Optional
    DOCKERHUB_PASSWORD = credentials('dockerhub-password')  // Optional
}
```

### Conditional Docker Hub Push
```groovy
stage('Push to Docker Hub (Optional)') {
    when {
        environment name: 'PUSH_TO_DOCKERHUB', value: 'true'
    }
    // ... push logic
}
```

## Setup Instructions

### 1. Local Registry (Automatic)
The pipeline automatically sets up a local registry. No manual configuration needed.

### 2. Docker Hub (Manual Setup)
1. Create Docker Hub credentials in Jenkins:
   - Go to Jenkins → Manage Jenkins → Credentials
   - Add credentials → Username with password
   - ID: `dockerhub-username`
   - ID: `dockerhub-password`

2. Set environment variable:
   ```bash
   PUSH_TO_DOCKERHUB=true
   ```

### 3. Registry Persistence
Local registry data is stored in `/var/lib/jenkins/registry` and persists between pipeline runs.

## Troubleshooting

### Registry Issues
```bash
# Check if registry is running
docker ps | grep registry

# Restart registry
docker restart registry

# Check registry logs
docker logs registry
```

### Docker Permission Issues
```bash
# Ensure Jenkins user is in docker group
sudo usermod -aG docker jenkins

# Fix socket permissions
sudo chmod 666 /var/run/docker.sock
```

### Image Not Found
```bash
# List local images
docker images

# List registry images
curl http://localhost:5000/v2/_catalog
```

## Best Practices

1. **Use Local Registry** for CI/CD pipelines
2. **Use Docker Hub** only when external sharing is needed
3. **Tag images** with build numbers for traceability
4. **Clean up** old images regularly
5. **Monitor registry** storage usage

## Security Considerations

- Local registry is insecure by default (HTTP)
- Docker Hub credentials are stored securely in Jenkins
- Images are cleaned up after pipeline completion
- Registry runs in isolated container

## Performance Tips

- Local registry eliminates network latency
- Registry data is persisted between builds
- Docker layer caching improves build times
- Consider registry cleanup for long-running systems
