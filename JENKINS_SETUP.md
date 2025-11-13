# Jenkins Deployment Pipeline Setup Guide

## Overview
This Jenkins pipeline automates the complete CI/CD workflow for deploying the Flask Docker application:
- Build Docker image
- Run tests
- Security scanning (optional)
- Push to Docker registry
- Deploy to target environment (Docker/Kubernetes/Cloud)
- Health checks

## Prerequisites

### 1. Jenkins Plugins Required
Install these plugins in Jenkins (Manage Jenkins → Manage Plugins):
- **Pipeline** (workflow-aggregator)
- **Docker Pipeline** 
- **Git plugin**
- **GitHub Branch Source** (for multibranch pipeline)
- **Docker plugin**
- **Kubernetes plugin** (if deploying to K8s)
- **Credentials Binding**
- **Pipeline: Stage View**

### 2. Jenkins Agent Requirements
The Jenkins agent must have:
- Docker installed and daemon running
- Git installed
- `curl` for health checks
- Access to Docker registry (Docker Hub, Azure Container Registry, etc.)
- (Optional) `kubectl` for Kubernetes deployments
- (Optional) `trivy` for security scanning

### 3. Docker Registry Access
- Docker Hub account OR private registry
- Authentication credentials

## Jenkins Configuration Steps

### Step 1: Add Credentials to Jenkins

#### A. Docker Registry Credentials
1. Go to Jenkins → Manage Jenkins → Manage Credentials
2. Click on "(global)" domain
3. Click "Add Credentials"
4. Select "Username with password"
   - **ID**: `dockerhub-credentials` (must match Jenkinsfile)
   - **Username**: Your Docker Hub username
   - **Password**: Docker Hub access token or password
   - **Description**: "Docker Hub credentials"
5. Click "OK"

#### B. Kubeconfig (if using Kubernetes)
1. Add Credentials → "Secret file"
   - **ID**: `kubeconfig`
   - **File**: Upload your kubeconfig file
   - **Description**: "Kubernetes config"

#### C. GitHub Token (optional, for branch protection)
1. Add Credentials → "Secret text"
   - **ID**: `github-token`
   - **Secret**: Your GitHub Personal Access Token
   - **Description**: "GitHub API token"

### Step 2: Create Jenkins Pipeline Job

#### Option A: Multibranch Pipeline (Recommended)
1. Jenkins Dashboard → New Item
2. Enter name: `flask-docker-app-pipeline`
3. Select "Multibranch Pipeline" → OK
4. Configure:
   - **Branch Sources** → Add source → GitHub
     - Owner: your-github-username
     - Repository: Docker-Project1
     - Credentials: (add GitHub credentials if repo is private)
     - Behaviors:
       - ✓ Discover branches
       - ✓ Discover pull requests from origin
   - **Build Configuration**:
     - Mode: by Jenkinsfile
     - Script Path: `Jenkinsfile`
   - **Scan Multibranch Pipeline Triggers**:
     - ✓ Periodically if not otherwise run
     - Interval: 1 minute (or setup GitHub webhook)
5. Save

#### Option B: Single Pipeline Job
1. New Item → Pipeline → OK
2. Configure:
   - **Pipeline** section:
     - Definition: "Pipeline script from SCM"
     - SCM: Git
     - Repository URL: your repo URL
     - Credentials: (if private)
     - Branch Specifier: `*/main`
     - Script Path: `Jenkinsfile`
3. Save

### Step 3: Configure Pipeline Environment Variables

Edit `Jenkinsfile` and update these variables:

```groovy
environment {
    DOCKER_IMAGE = 'your-dockerhub-username/flask-app'  // Update with your Docker Hub username
    DOCKER_REGISTRY = 'docker.io'  // Or your private registry
    DOCKER_REGISTRY_CREDENTIAL = 'dockerhub-credentials'
    APP_PORT = '5000'  // Or your desired port
}
```

### Step 4: Setup GitHub Branch Protection (Optional)

To prevent merging PRs until Jenkins pipeline passes:

1. GitHub repo → Settings → Branches
2. Add rule for `main` branch:
   - ✓ Require a pull request before merging
   - ✓ Require status checks to pass before merging
     - Add required check: `continuous-integration/jenkins/pr-merge` (or the check name from Jenkins)
   - ✓ Require branches to be up to date before merging
3. Save changes

## Deployment Options

The pipeline supports multiple deployment targets. Choose one by uncommenting the relevant function call in the `Deploy` stage:

### Option 1: Local Docker (Default)
Deploys to the Jenkins agent's Docker host:
```groovy
deployToDocker()  // Already active
```
Access: `http://jenkins-host:5000`

### Option 2: Kubernetes
Deploy to a Kubernetes cluster:
```groovy
deployToKubernetes()  // Uncomment in Jenkinsfile
```

Prerequisites:
- Add kubeconfig credential to Jenkins
- Apply K8s resources first:
  ```bash
  kubectl apply -f k8s-deployment.yaml
  ```

### Option 3: Docker Swarm
Deploy to Docker Swarm cluster:
```groovy
deployToSwarm()  // Uncomment in Jenkinsfile
```

Prerequisites:
- Initialize Docker Swarm: `docker swarm init`

### Option 4: Cloud Services
Deploy to AWS ECS, Azure Container Instances, Google Cloud Run, etc.:
```groovy
deployToCloud()  // Uncomment and customize
```

## Pipeline Stages Explained

| Stage | Purpose | Actions |
|-------|---------|---------|
| **Checkout** | Get source code | Clones Git repository |
| **Build Docker Image** | Create container | Builds image from Dockerfile |
| **Test** | Validate code | Runs tests inside container |
| **Security Scan** | Find vulnerabilities | Scans image with Trivy (optional) |
| **Push to Registry** | Store image | Pushes to Docker Hub/registry |
| **Deploy** | Release to environment | Deploys based on branch |
| **Health Check** | Verify deployment | Checks if app responds |

## Branch-Based Deployment Strategy

- **main branch** → Deploys to **production** environment
- **feature/* branches** → Deploys to **staging** environment
- **Pull requests** → Build + test only (no deployment)

## Testing the Pipeline

### 1. Trigger a Build
Push code to your repository:
```powershell
git add .
git commit -m "Add Jenkins pipeline"
git push origin main
```

### 2. Monitor Build
- Go to Jenkins → Your pipeline job
- Click on the build number
- View "Console Output" for logs

### 3. Verify Deployment
After successful build:
```powershell
# Check running container
docker ps | findstr flask-docker-app

# Test the application
curl http://localhost:5000
```

Expected output: `Hello, Docker!`

## Common Issues & Solutions

### Issue 1: Docker permission denied
**Error**: `permission denied while trying to connect to Docker daemon`

**Solution**: Add Jenkins user to docker group:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Issue 2: Cannot push to Docker registry
**Error**: `denied: requested access to the resource is denied`

**Solution**:
- Verify Docker Hub credentials in Jenkins
- Update `DOCKER_IMAGE` to include your username: `your-username/flask-app`
- Login manually on agent: `docker login`

### Issue 3: Port already in use
**Error**: `Bind for 0.0.0.0:5000 failed: port is already allocated`

**Solution**: Stop existing container or change port:
```powershell
docker stop flask-docker-app-production
# Or change APP_PORT in Jenkinsfile
```

### Issue 4: Kubernetes deployment not found
**Error**: `Error from server (NotFound): deployments.apps "flask-docker-app" not found`

**Solution**: Create deployment first:
```bash
kubectl apply -f k8s-deployment.yaml
```

## Advanced Configuration

### Add Slack Notifications
1. Install Slack Notification plugin
2. Configure in Jenkinsfile:
```groovy
post {
    success {
        slackSend(color: 'good', message: "Deploy succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
    }
}
```

### Enable Security Scanning with Trivy
Install Trivy on Jenkins agent:
```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
```

Uncomment in Jenkinsfile `Security Scan` stage.

### Add Database/Redis Services
Update `docker-compose.yml` to include additional services:
```yaml
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
```

## Maintenance

### Clean Old Docker Images
Jenkins automatically prunes images in the `post` block. Manual cleanup:
```powershell
docker system prune -a -f
```

### View Logs
```powershell
docker logs flask-docker-app-production
```

### Rollback Deployment
```powershell
# Redeploy previous build
docker run -d --name flask-docker-app-production -p 5000:5000 flask-app:<previous-build-number>
```

## Next Steps

1. **Setup monitoring**: Add Prometheus/Grafana for metrics
2. **Configure auto-scaling**: Use Kubernetes HPA or Docker Swarm scaling
3. **Add staging environment**: Create separate namespace/cluster
4. **Implement blue-green deployment**: Zero-downtime deployments
5. **Add integration tests**: Test against deployed application

## Support

For issues:
- Check Jenkins console logs
- Review Docker logs: `docker logs <container-name>`
- Verify credentials and permissions
- Ensure all prerequisites are installed

---

**Project Structure:**
```
Docker-Project1/
├── app.py                    # Flask application
├── Dockerfile                # Docker image definition
├── requirements.txt          # Python dependencies
├── Jenkinsfile              # CI/CD pipeline definition
├── docker-compose.yml       # Docker Compose config
├── k8s-deployment.yaml      # Kubernetes deployment
└── JENKINS_SETUP.md         # This file
```
