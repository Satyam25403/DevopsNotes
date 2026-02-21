# Jenkins - CI/CD Automation Server

Complete guide to Jenkins for continuous integration and continuous deployment pipelines.

## Table of Contents
- [What is CI/CD](#what-is-cicd)
- [Jenkins Overview](#jenkins-overview)
- [Installation and Setup](#installation-and-setup)
- [Jenkins Configuration](#jenkins-configuration)
- [Creating Jobs](#creating-jobs)
- [Pipeline as Code](#pipeline-as-code)
- [Docker Integration](#docker-integration)
- [Production Pipeline Example](#production-pipeline-example)

---

## What is CI/CD?

### Continuous Integration (CI)

**Taking code and making it ready to ship:**
- Pull code from repository
- Install dependencies
- Set up environment
- Run tests
- Build artifacts
- Package application

### Continuous Delivery/Deployment (CD)

**Delivering packaged code to production:**
- Deploy to staging/production
- Make application live
- Serve end users
- Automated rollbacks if needed

### Why CI/CD?

**Before CI/CD:**
- Manual deployments taking days/weeks
- Late bug discovery (costly to fix)
- Merge conflicts from siloed work
- Human error in deployment
- Risky releases

**After CI/CD:**
- Deployments in hours, not days
- Early bug detection (cheaper to fix)
- Continuous collaboration
- Automated, consistent deployments
- Quick rollbacks
- Faster customer feedback

### Real-World Impact

**Faster Releases:**
- Ship features in hours
- Quick bug fixes
- Agile response to market

**Early Bug Detection:**
- Automated tests on every commit
- Catch errors before production
- Reduce debugging costs

**Improved Collaboration:**
- Developers, testers, ops in sync
- Frequent commits prevent conflicts
- Shared responsibility

**Reliable Deployments:**
- Eliminate manual errors
- Consistent environments
- Easy rollbacks

---

## Jenkins Overview

**Jenkins** is an open-source automation server for CI/CD pipelines.

### Types of CI/CD Tools

**1. Self-Hosted:**
- Run on your own infrastructure
- Full control over resources
- Self-managed
- Example: **Jenkins**

**2. VCS-Embedded:**
- Built into version control
- No separate infrastructure
- Managed by VCS provider
- Examples: GitHub Actions, GitLab CI, Bitbucket Pipelines

**3. External Managed:**
- Third-party service
- Not part of VCS
- Fully managed
- Examples: CircleCI, Travis CI

### Why Jenkins?

**✅ Open Source:**
- Free to use
- Large community
- Extensive plugins

**✅ Flexible:**
- Any language, any platform
- Customizable pipelines
- Rich ecosystem

**✅ Powerful:**
- Parallel execution
- Distributed builds
- Advanced scheduling

**✅ Integrations:**
- Git, Docker, Kubernetes
- AWS, GCP, Azure
- Slack, email notifications

---

## Installation and Setup

### EC2/Server Installation

**Prerequisites:**
```bash
# Check Java (required)
java -version

# Update system
sudo apt update
```

**Install Jenkins:**
```bash
# Install Java
sudo apt install openjdk-11-jdk -y

# Import Jenkins key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Add Jenkins repository
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update and install
sudo apt update
sudo apt install jenkins -y

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check status
sudo systemctl status jenkins

# Get initial password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Access Jenkins:**
```
http://<ec2-ip>:8080
```

### Docker Installation

```bash
docker run -d -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  --name jenkins \
  jenkins/jenkins:lts
```

### Local Installation

**Linux:**
```bash
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt install jenkins
```

**macOS:**
```bash
brew install jenkins-lts
brew services start jenkins-lts
```

**Windows:**
- Download installer from jenkins.io
- Run installer
- Access at http://localhost:8080

---

## Jenkins Configuration

### Initial Setup

**1. Unlock Jenkins:**
```bash
# Get password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
- Paste in browser at http://localhost:8080

**2. Install Plugins:**
- **Recommended:**
  - Git
  - Pipeline
  - Docker
  - GitHub Integration
  - SSH Agent
  - Credentials Binding

**3. Create Admin User:**
- Username
- Password
- Full name
- Email

**4. Configure Instance:**
- Jenkins URL
- System settings

### Give Jenkins Docker Access

**Critical step for Docker builds:**
```bash
# Add Jenkins user to docker group
sudo usermod -aG docker jenkins

# Restart Jenkins
sudo systemctl restart jenkins

# Verify
sudo -u jenkins docker ps
```

### Install Additional Plugins

**Manage Jenkins → Plugins:**
- Docker Pipeline
- Kubernetes
- AWS Steps
- Slack Notification
- Blue Ocean (modern UI)

---

## Creating Jobs

### Freestyle Project

**1. New Item:**
- Enter name
- Select "Freestyle project"
- Click OK

**2. Source Code Management:**
- Git
- Repository URL: `https://github.com/user/repo.git`
- Credentials: Add GitHub username/password or token

**3. Build Triggers:**
- **Poll SCM:** Check for changes
  - Schedule: `H/5 * * * *` (every 5 minutes)
  - Cron syntax: `* * * * *` (every minute)

- **GitHub hook trigger:** Webhook from GitHub
- **Build periodically:** Time-based

**4. Build Steps:**
- Execute shell:
```bash
docker stop edward || true
docker rm edward || true
docker build -t edward .
docker run --name edward -d -p 9000:9000 edward
```

**5. Save and Build:**
- Click "Save"
- Click "Build Now"

### Build Triggers (Cron Syntax)

```
* * * * *
│ │ │ │ │
│ │ │ │ └── Day of week (0-7, Sun-Sat)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)
```

**Examples:**
```
H/5 * * * *      # Every 5 minutes
0 0 * * *        # Daily at midnight
0 */4 * * *      # Every 4 hours
0 0 * * 0        # Weekly on Sunday
```

---

## Pipeline as Code

### Jenkinsfile

**Create in repository root:**
```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "myapp"
        DOCKER_PORT = "9000"
    }
    
    stages {
        stage('Clone') {
            steps {
                git credentialsId: 'github-creds',
                    url: 'https://github.com/user/repo.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE} .'
            }
        }
        
        stage('Test') {
            steps {
                sh 'docker run --rm ${DOCKER_IMAGE} npm test'
            }
        }
        
        stage('Deploy') {
            steps {
                sh '''
                    docker stop ${DOCKER_IMAGE} || true
                    docker rm ${DOCKER_IMAGE} || true
                    docker run -d \
                        --name ${DOCKER_IMAGE} \
                        -p ${DOCKER_PORT}:${DOCKER_PORT} \
                        ${DOCKER_IMAGE}
                '''
            }
        }
    }
    
    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}
```

### Pipeline Job

**1. New Item:**
- Pipeline project

**2. Pipeline Definition:**
- **Pipeline script from SCM:**
  - SCM: Git
  - Repository URL
  - Script Path: `Jenkinsfile`

**3. Build Triggers:**
- Same as Freestyle

---

## Docker Integration

### Complete Docker Pipeline

**Jenkinsfile:**
```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE_NAME = "edward"
        DOCKER_PORT = "9000"
        DOCKER_REGISTRY = "dockerhub.io"
        REGISTRY_CREDENTIALS = "dockerhub-creds"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'github-creds',
                    url: 'https://github.com/user/private-repo.git',
                    branch: 'main'
            }
        }
        
        stage('Stop Existing Container') {
            steps {
                sh '''
                    docker stop ${DOCKER_IMAGE_NAME} || true
                    docker rm ${DOCKER_IMAGE_NAME} || true
                '''
            }
        }
        
        stage('Build Image') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE_NAME}:latest .'
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    docker run --rm ${DOCKER_IMAGE_NAME}:latest npm test
                '''
            }
        }
        
        stage('Push to Registry') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', REGISTRY_CREDENTIALS) {
                        sh 'docker tag ${DOCKER_IMAGE_NAME}:latest username/${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}'
                        sh 'docker push username/${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}'
                    }
                }
            }
        }
        
        stage('Deploy') {
            steps {
                sh '''
                    docker run -d \
                        --name ${DOCKER_IMAGE_NAME} \
                        -p ${DOCKER_PORT}:${DOCKER_PORT} \
                        --restart unless-stopped \
                        ${DOCKER_IMAGE_NAME}:latest
                '''
            }
        }
        
        stage('Cleanup') {
            steps {
                sh 'docker image prune -f'
            }
        }
    }
    
    post {
        success {
            echo "✅ App deployed at http://<ec2-ip>:${DOCKER_PORT}"
        }
        failure {
            echo "❌ Deployment failed. Check logs."
        }
        always {
            sh 'docker logout'
        }
    }
}
```

---

## Production Pipeline Example

### Complete Workflow

**Project Structure:**
```
repo/
├── Dockerfile
├── Jenkinsfile
├── src/
├── tests/
├── package.json
└── .env.example
```

**Jenkinsfile:**
```groovy
pipeline {
    agent any
    
    environment {
        APP_NAME = "myapp"
        DOCKER_IMAGE = "username/myapp"
        DEPLOY_PORT = "9000"
        SLACK_CHANNEL = "#deployments"
    }
    
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Deployment environment'
        )
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Run tests before deployment'
        )
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Environment Setup') {
            steps {
                script {
                    if (params.ENVIRONMENT == 'prod') {
                        DEPLOY_PORT = '80'
                    }
                }
            }
        }
        
        stage('Build') {
            steps {
                sh """
                    docker build \
                        --build-arg ENV=${params.ENVIRONMENT} \
                        -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                        -t ${DOCKER_IMAGE}:latest \
                        .
                """
            }
        }
        
        stage('Test') {
            when {
                expression { params.RUN_TESTS }
            }
            steps {
                sh """
                    docker run --rm \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                        npm test
                """
            }
        }
        
        stage('Security Scan') {
            steps {
                sh "docker scan ${DOCKER_IMAGE}:${BUILD_NUMBER} || true"
            }
        }
        
        stage('Push to Registry') {
            steps {
                script {
                    docker.withRegistry('', 'dockerhub-creds') {
                        sh "docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                        sh "docker push ${DOCKER_IMAGE}:latest"
                    }
                }
            }
        }
        
        stage('Deploy') {
            steps {
                sh """
                    # Stop old container
                    docker stop ${APP_NAME} || true
                    docker rm ${APP_NAME} || true
                    
                    # Run new container
                    docker run -d \
                        --name ${APP_NAME} \
                        -p ${DEPLOY_PORT}:${DEPLOY_PORT} \
                        -e NODE_ENV=${params.ENVIRONMENT} \
                        --restart unless-stopped \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER}
                """
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    sleep 10
                    sh """
                        curl -f http://localhost:${DEPLOY_PORT}/health || exit 1
                    """
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                sh """
                    docker image prune -f
                    docker container prune -f
                """
            }
        }
    }
    
    post {
        success {
            slackSend(
                channel: SLACK_CHANNEL,
                color: 'good',
                message: "✅ ${APP_NAME} deployed to ${params.ENVIRONMENT}
Build: ${BUILD_NUMBER}
URL: http://<server-ip>:${DEPLOY_PORT}"
            )
        }
        
        failure {
            slackSend(
                channel: SLACK_CHANNEL,
                color: 'danger',
                message: "❌ ${APP_NAME} deployment failed
Build: ${BUILD_NUMBER}
Environment: ${params.ENVIRONMENT}"
            )
        }
        
        always {
            cleanWs()
        }
    }
}
```

### Credentials Setup

**Manage Jenkins → Credentials:**

**1. GitHub Credentials:**
- Kind: Username with password
- Username: GitHub username
- Password: Personal Access Token
- ID: `github-creds`

**2. Docker Hub Credentials:**
- Kind: Username with password
- Username: Docker Hub username
- Password: Docker Hub password/token
- ID: `dockerhub-creds`

**3. SSH Key:**
- Kind: SSH Username with private key
- ID: `ec2-ssh-key`
- Private key: Paste EC2 key

### Webhook Setup

**GitHub:**
1. Repository → Settings → Webhooks
2. Add webhook:
   - URL: `http://<jenkins-url>/github-webhook/`
   - Content type: `application/json`
   - Events: Push events

**Jenkins:**
- Job → Configure → Build Triggers
- ✅ GitHub hook trigger for GITScm polling

---

## Quick Reference

### Essential Commands

```bash
# Start/Stop Jenkins
sudo systemctl start jenkins
sudo systemctl stop jenkins
sudo systemctl restart jenkins

# Check status
sudo systemctl status jenkins

# View logs
sudo journalctl -u jenkins -f

# Initial password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Pipeline Stages

```groovy
stages {
    stage('Checkout') { }
    stage('Build') { }
    stage('Test') { }
    stage('Deploy') { }
}
```

### Common Post Actions

```groovy
post {
    success { }
    failure { }
    always { }
    unstable { }
}
```

---

This guide covers Jenkins CI/CD automation from installation to production Docker deployment pipelines.