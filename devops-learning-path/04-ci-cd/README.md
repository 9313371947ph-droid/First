# Module 04: CI/CD Pipelines - Continuous Integration & Continuous Deployment

## 📚 Table of Contents
1. [What is CI/CD?](#what-is-cicd)
2. [CI/CD Pipeline Stages](#cicd-pipeline-stages)
3. [Jenkins - The Classic CI/CD Tool](#jenkins---the-classic-cicd-tool)
4. [GitHub Actions - Modern CI/CD](#github-actions---modern-cicd)
5. [Pipeline as Code](#pipeline-as-code)
6. [Best Practices](#best-practices)
7. [Hands-on Labs](#hands-on-labs)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Knowledge Check](#knowledge-check)

---

## What is CI/CD?

### Continuous Integration (CI)
**Definition**: A development practice where developers frequently merge code changes into a central repository, followed by automated builds and tests.

**Key Benefits**:
- ✅ Early bug detection
- ✅ Reduced integration problems
- ✅ Faster feedback loops
- ✅ Improved code quality
- ✅ Increased team collaboration

**CI Workflow**:
```
Developer commits code 
    ↓
Code pushed to repository 
    ↓
Automated build triggered 
    ↓
Automated tests run 
    ↓
Feedback provided (pass/fail)
```

### Continuous Delivery (CD)
**Definition**: An extension of CI that automatically deploys all code changes to a testing or staging environment after the build stage.

**Key Characteristics**:
- ✅ Automated deployment to staging
- ✅ Manual approval for production
- ✅ Ready to deploy at any time
- ✅ Consistent deployment process

### Continuous Deployment
**Definition**: The highest level of automation where every change that passes all stages of your production pipeline is released to customers automatically.

**Key Characteristics**:
- ✅ Fully automated production deployments
- ✅ No manual intervention
- ✅ Multiple deployments per day
- ✅ Requires excellent test coverage

**CI/CD Spectrum**:
```
CI → Continuous Delivery → Continuous Deployment
     (Manual Approval)      (Fully Automated)
```

---

## CI/CD Pipeline Stages

### Typical Pipeline Flow

```mermaid
graph LR
    A[Code Commit] --> B[Build]
    B --> C[Unit Tests]
    C --> D[Code Quality]
    D --> E[Integration Tests]
    E --> D{Security Scan}
    D --> G[Deploy to Staging]
    G --> H[UAT Tests]
    H --> I[Manual Approval]
    I --> J[Deploy to Production]
    J --> K[Monitoring]
```

### Stage Breakdown

#### 1. **Source Stage**
- Triggered by code commit/push
- Pulls latest code from repository
- Supports multiple branches

#### 2. **Build Stage**
- Compiles source code
- Resolves dependencies
- Creates executable artifacts
- Docker image creation (if applicable)

#### 3. **Test Stage**
- **Unit Tests**: Test individual components
- **Integration Tests**: Test component interactions
- **Functional Tests**: Test business requirements
- **Performance Tests**: Load and stress testing
- **Security Tests**: Vulnerability scanning

#### 4. **Code Quality Stage**
- Static code analysis
- Code coverage reports
- Security vulnerability checks
- Compliance validation

#### 5. **Deploy to Staging**
- Deploy to test environment
- Environment configuration
- Database migrations
- Service restarts

#### 6. **User Acceptance Testing (UAT)**
- Manual testing by QA team
- Business stakeholder validation
- Performance validation

#### 7. **Deploy to Production**
- Blue-Green deployment OR
- Canary deployment OR
- Rolling update

#### 8. **Post-Deployment**
- Health checks
- Monitoring setup
- Rollback capability
- Notification alerts

---

## Jenkins - The Classic CI/CD Tool

### What is Jenkins?

**Jenkins** is an open-source automation server written in Java. It's one of the most popular CI/CD tools with over 1,800 plugins.

### Key Features
- ✅ Open source and free
- ✅ Massive plugin ecosystem
- ✅ Distributed builds
- ✅ Pipeline as Code support
- ✅ Easy configuration via web UI
- ✅ Strong community support

### Installation

#### Method 1: Docker (Recommended for Learning)
```bash
# Pull Jenkins LTS image
docker pull jenkins/jenkins:lts

# Run Jenkins container
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts

# Get initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

#### Method 2: Ubuntu/Debian
```bash
# Add Jenkins repository
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

# Install Jenkins
sudo apt update
sudo apt install jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Access at http://localhost:8080
```

### Jenkins Architecture

```
┌─────────────┐
│   Master    │ ← Controls workflow, schedules jobs, monitors agents
│  (Controller)│
└──────┬──────┘
       │
       ├──────────────┐
       │              │
┌──────▼──────┐ ┌─────▼──────┐
│   Agent 1   │ │   Agent 2  │ ← Execute build jobs
└─────────────┘ └────────────┘
```

### Jenkins Pipeline Concepts

#### Pipeline Syntax Types

1. **Declarative Pipeline** (Recommended)
   - Structured and opinionated
   - Easier to read and write
   - Better for most use cases

2. **Scripted Pipeline**
   - More flexible and powerful
   - Uses Groovy scripting
   - Steeper learning curve

### Declarative Pipeline Example

```groovy
pipeline {
    agent any
    
    environment {
        APP_NAME = "my-app"
        DOCKER_REGISTRY = "docker.io/myuser"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'npm install'
                sh 'npm run build'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'npm test'
            }
            post {
                always {
                    junit 'reports/*.xml'
                }
            }
        }
        
        stage('Code Quality') {
            steps {
                echo 'Running SonarQube analysis...'
                sh 'npm run lint'
            }
        }
        
        stage('Docker Build') {
            steps {
                echo 'Building Docker image...'
                script {
                    docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_ID}")
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Deploying to staging...'
                sh 'kubectl apply -f k8s/staging/'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                echo 'Deploying to production...'
                sh 'kubectl apply -f k8s/production/'
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
            cleanWs()
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
            // Send notification
        }
    }
}
```

### Jenkinsfile Best Practices

```groovy
// ✅ GOOD: Using environment variables
environment {
    DB_HOST = credentials('db-host')
    API_KEY = credentials('api-key')
}

// ✅ GOOD: Parallel stages for faster execution
stage('Tests') {
    parallel {
        stage('Unit Tests') {
            steps { sh 'npm test:unit' }
        }
        stage('Integration Tests') {
            steps { sh 'npm test:integration' }
        }
        stage('E2E Tests') {
            steps { sh 'npm test:e2e' }
        }
    }
}

// ✅ GOOD: Error handling
post {
    failure {
        slackSend channel: '#alerts', 
                  color: 'danger', 
                  message: "Build failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
    }
}

// ❌ BAD: Hardcoded credentials
sh 'deploy --password=mysecret123'

// ✅ GOOD: Using credentials binding
withCredentials([string(credentialsId: 'db-password', variable: 'DB_PASS')]) {
    sh 'deploy --password=$DB_PASS'
}
```

### Essential Jenkins Plugins

| Plugin | Purpose |
|--------|---------|
| Pipeline | Pipeline as Code support |
| Git | Git integration |
| GitHub | GitHub integration |
| Docker Pipeline | Docker operations in pipelines |
| Kubernetes | Kubernetes integration |
| SonarQube | Code quality analysis |
| Slack Notification | Team notifications |
| Blue Ocean | Modern UI |
| Build Timeout | Prevent hanging builds |
| Credentials Binding | Secure credential management |

---

## GitHub Actions - Modern CI/CD

### What is GitHub Actions?

**GitHub Actions** is a CI/CD platform built directly into GitHub. It allows you to automate workflows directly in your repository.

### Key Advantages
- ✅ Native GitHub integration
- ✅ Free for public repositories
- ✅ Generous free tier for private repos
- ✅ YAML-based configuration
- ✅ Huge marketplace of actions
- ✅ No infrastructure management

### Core Concepts

#### 1. **Workflow**
- Automated process defined in YAML
- Stored in `.github/workflows/`
- Can have multiple workflows per repo

#### 2. **Event**
- Triggers the workflow
- Examples: push, pull_request, schedule, manual

#### 3. **Job**
- Set of steps that execute on same runner
- Jobs run in parallel by default
- Can define dependencies between jobs

#### 4. **Step**
- Individual task in a job
- Can be action or shell command
- Executes sequentially

#### 5. **Action**
- Reusable unit of code
- Can be written by anyone
- Available in GitHub Marketplace

#### 6. **Runner**
- Server that executes workflows
- GitHub-hosted or self-hosted
- Different OS options available

### Basic Workflow Example

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run unit tests
        run: npm test
      
      - name: Build application
        run: npm run build
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.node-version }}
          path: dist/

  security-scan:
    runs-on: ubuntu-latest
    needs: build-and-test
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Run security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  deploy-staging:
    runs-on: ubuntu-latest
    needs: [build-and-test, security-scan]
    if: github.ref == 'refs/heads/develop'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Deploy to staging
        run: |
          echo "Deploying to staging environment"
          # Add deployment commands here

  deploy-production:
    runs-on: ubuntu-latest
    needs: [build-and-test, security-scan]
    if: github.ref == 'refs/heads/main'
    
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Deploy to production
        run: |
          echo "Deploying to production"
          # Add deployment commands here
```

### Advanced GitHub Actions Features

#### 1. **Environment Protection Rules**
```yaml
deploy-production:
  runs-on: ubuntu-latest
  
  environment: 
    name: production
    url: https://myapp.com
  
  steps:
    - name: Deploy
      run: ./deploy.sh
```

#### 2. **Manual Triggers (Workflow Dispatch)**
```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      
      version:
        description: 'Version to deploy'
        required: false
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          echo "Deploying version ${{ inputs.version }} to ${{ inputs.environment }}"
```

#### 3. **Scheduled Workflows (Cron)**
```yaml
on:
  schedule:
    # Run every day at 2 AM UTC
    - cron: '0 2 * * *'
    # Run every Monday at 9 AM UTC
    - cron: '0 9 * * 1'

jobs:
  nightly-build:
    runs-on: ubuntu-latest
    steps:
      - name: Run nightly tasks
        run: ./nightly-build.sh
```

#### 4. **Reusable Workflows**
```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      version:
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          echo "Deploying ${{ inputs.version }} to ${{ inputs.environment }}"
```

#### 5. **Composite Actions**
```yaml
# action.yml
name: 'Setup and Build'
description: 'Setup environment and build application'
inputs:
  node-version:
    description: 'Node.js version'
    required: true
    default: '18'

runs:
  using: "composite"
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
    
    - name: Install dependencies
      shell: bash
      run: npm ci
    
    - name: Build
      shell: bash
      run: npm run build
```

### GitHub Actions vs Jenkins Comparison

| Feature | GitHub Actions | Jenkins |
|---------|---------------|---------|
| **Setup** | Zero setup, built-in | Requires installation |
| **Configuration** | YAML in repo | UI + Jenkinsfile |
| **Cost** | Free for public, generous free tier | Free (self-hosted costs) |
| **Scalability** | Auto-scaling runners | Manual agent management |
| **Plugins** | Marketplace actions | 1,800+ plugins |
| **Learning Curve** | Low | Medium-High |
| **Customization** | Limited | Extensive |
| **Best For** | GitHub-native projects | Complex, custom workflows |

---

## Pipeline as Code

### What is Pipeline as Code?

**Pipeline as Code** is the practice of defining your CI/CD pipeline configuration in version-controlled files alongside your application code.

### Benefits
- ✅ Version control for pipelines
- ✅ Code review for pipeline changes
- ✅ Consistency across environments
- ✅ Disaster recovery (recreate from code)
- ✅ Documentation through code
- ✅ Reusability and templating

### Jenkinsfile Structure

```groovy
// Jenkinsfile
@Library('jenkins-library@v1') _

pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: maven
                    image: maven:3.8-openjdk-17
                    command:
                    - cat
                    tty: true
                  - name: docker
                    image: docker:20.10-dind
                    securityContext:
                      privileged: true
            '''
        }
    }
    
    options {
        timeouts {
            activity timeout: '1 hour'
            stage 'Build', timeout: '20 minutes'
        }
        retry(2)
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }
    
    environment {
        DOCKER_REGISTRY = credentials('docker-registry-url')
        SONAR_TOKEN = credentials('sonarqube-token')
    }
    
    parameters {
        string(name: 'VERSION', defaultValue: 'latest', description: 'Version to build')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip tests')
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: 'Target environment')
    }
    
    stages {
        stage('Initialize') {
            steps {
                script {
                    env.BUILD_VERSION = params.VERSION ?: env.BUILD_ID
                }
            }
        }
        
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests=${params.SKIP_TESTS}'
                }
            }
        }
        
        stage('Test') {
            when {
                expression { return !params.SKIP_TESTS }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('maven') {
                            sh 'mvn test'
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        container('maven') {
                            sh 'mvn verify -Pintegration-tests'
                        }
                    }
                }
            }
        }
        
        stage('Code Quality') {
            steps {
                container('maven') {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn sonar:sonar -Dsonar.projectKey=${JOB_NAME}'
                    }
                }
            }
        }
        
        stage('Docker Build & Push') {
            steps {
                container('docker') {
                    script {
                        docker.withRegistry(env.DOCKER_REGISTRY, 'docker-credentials') {
                            def image = docker.build("${APP_NAME}:${BUILD_VERSION}")
                            image.push()
                        }
                    }
                }
            }
        }
        
        stage('Deploy') {
            when {
                expression { return params.ENVIRONMENT != null }
            }
            steps {
                script {
                    if (params.ENVIRONMENT == 'production') {
                        input message: 'Deploy to production?', ok: 'Deploy'
                    }
                    sh "./deploy.sh ${params.ENVIRONMENT} ${BUILD_VERSION}"
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
            archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
        }
        success {
            slackSend channel: '#deployments', 
                      color: 'good', 
                      message: ":white_check_mark: Build ${env.JOB_NAME} #${env.BUILD_NUMBER} succeeded!"
        }
        failure {
            slackSend channel: '#alerts', 
                      color: 'danger', 
                      message: ":x: Build ${env.JOB_NAME} #${env.BUILD_NUMBER} failed!"
        }
    }
}
```

### GitHub Actions Workflow Structure

```yaml
# .github/workflows/full-pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      skip_tests:
        description: 'Skip tests'
        required: false
        default: 'false'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

permissions:
  contents: read
  packages: write
  security-events: write

jobs:
  validate:
    name: Validate
    runs-on: ubuntu-latest
    outputs:
      should_build: ${{ steps.check.outputs.should_build }}
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Check if build needed
        id: check
        run: |
          # Only build if code changed
          echo "should_build=true" >> $GITHUB_OUTPUT

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: validate
    if: needs.validate.outputs.should_build == 'true'
    
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup
        uses: ./.github/actions/setup
      
      - name: Build
        run: npm run build
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.os }}
          path: dist/
          retention-days: 7

  test:
    name: Test Suite
    runs-on: ubuntu-latest
    needs: build
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Download artifacts
        uses: actions/download-artifact@v4
      
      - name: Run unit tests
        run: npm test
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: false

  security:
    name: Security Scans
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Run SAST
        uses: github/codeql-action/init@v2
        with:
          languages: javascript
      
      - name: Analyze
        uses: github/codeql-action/analyze@v2
      
      - name: Dependency scan
        uses: snyk/actions/node@master
        continue-on-error: true
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      - name: Container scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'ghcr.io/${{ github.repository }}:latest'
          format: 'sarif'
          output: 'trivy-results.sarif'

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
      - name: Deploy
        run: |
          echo "Deploying to staging"
          # kubectl apply -f k8s/staging/

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [test, security]
    if: startsWith(github.ref, 'refs/tags/v')
    environment: production
    
    steps:
      - name: Deploy
        run: |
          echo "Deploying ${{ github.ref_name }} to production"
          # kubectl apply -f k8s/production/
```

---

## Best Practices

### 1. **Keep Pipelines Fast**
```yaml
# ✅ GOOD: Parallel execution
jobs:
  test:
    strategy:
      matrix:
        test-type: [unit, integration, e2e]
    steps:
      - run: npm run test:${{ matrix.test-type }}

# ❌ BAD: Sequential execution
steps:
  - run: npm run test:unit
  - run: npm run test:integration
  - run: npm run test:e2e
```

### 2. **Use Caching**
```yaml
# ✅ GOOD: Cache dependencies
- name: Cache node modules
  uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# ❌ BAD: Download dependencies every time
- run: npm install
```

### 3. **Secure Secrets Management**
```yaml
# ✅ GOOD: Use secrets
env:
  API_KEY: ${{ secrets.API_KEY }}
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

# ❌ BAD: Hardcoded values
env:
  API_KEY: abc123xyz
  DB_PASSWORD: mypassword123
```

### 4. **Implement Proper Error Handling**
```groovy
// ✅ GOOD: Error handling with notifications
post {
    failure {
        slackSend channel: '#alerts', message: "Build failed!"
        emailext subject: "Build Failed", body: "Check: ${env.BUILD_URL}"
    }
    unstable {
        echo "Tests are unstable, investigating..."
    }
}
```

### 5. **Use Meaningful Stage Names**
```groovy
// ✅ GOOD: Descriptive names
stage('Run Unit Tests with Coverage')
stage('Build Docker Image v1.2.3')
stage('Deploy to Production US-East')

// ❌ BAD: Vague names
stage('Test')
stage('Build')
stage('Deploy')
```

### 6. **Implement Pipeline Testing**
```groovy
// Use Jenkins Pipeline Unit Testing framework
@Test
void 'should build successfully'() {
    runScript('Jenkinsfile')
    assertBuildStatus(Result.SUCCESS)
}
```

### 7. **Version Your Pipelines**
```yaml
# ✅ GOOD: Pin action versions
uses: actions/checkout@v4
uses: actions/setup-node@v4

# ❌ BAD: Use latest (can break)
uses: actions/checkout@latest
uses: actions/checkout@main
```

### 8. **Optimize Resource Usage**
```yaml
# ✅ GOOD: Specify exact runner needs
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
# ❌ BAD: Wasteful configuration
jobs:
  build:
    runs-on: self-hosted
    # No timeout specified
```

### 9. **Document Your Pipelines**
```yaml
# ✅ GOOD: Well-documented workflow
name: CI/CD Pipeline
# This workflow handles:
# - Building and testing on every PR
# - Deploying to staging on develop branch
# - Deploying to production on version tags
# Owner: Platform Team
# Last Updated: 2024-01-15

on:
  # ... rest of workflow
```

### 10. **Monitor Pipeline Metrics**
Track these KPIs:
- Build success rate
- Average build time
- Deployment frequency
- Mean time to recovery (MTTR)
- Change failure rate

---

## Hands-on Labs

### Lab 1: Create Your First Jenkins Pipeline

**Objective**: Set up Jenkins and create a simple CI pipeline

**Prerequisites**:
- Docker installed
- Git repository with sample code

**Steps**:

1. **Start Jenkins**
```bash
docker run -d \
  --name jenkins-lab \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins-data:/var/jenkins_home \
  jenkins/jenkins:lts
```

2. **Access Jenkins**
- Open browser: http://localhost:8080
- Get admin password: `docker exec jenkins-lab cat /var/jenkins_home/secrets/initialAdminPassword`
- Complete setup wizard
- Install suggested plugins

3. **Create Pipeline Job**
- Click "New Item"
- Enter name: "my-first-pipeline"
- Select "Pipeline"
- Click "OK"

4. **Configure Pipeline**
- In "Pipeline" section, select "Pipeline script"
- Paste this code:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Hello') {
            steps {
                echo 'Hello World!'
                echo "Build number: ${env.BUILD_NUMBER}"
                echo "Workspace: ${env.WORKSPACE}"
            }
        }
        
        stage('Clone Repository') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/octocat/Hello-World.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'ls -la'
                sh 'cat README'
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
        }
    }
}
```

5. **Run Pipeline**
- Click "Build Now"
- Watch console output
- Verify all stages pass

### Lab 2: GitHub Actions CI Pipeline

**Objective**: Create a complete CI pipeline with GitHub Actions

**Prerequisites**:
- GitHub account
- Repository with Node.js/Python/Java code

**Steps**:

1. **Create Workflow File**
```bash
# In your repository
mkdir -p .github/workflows
touch .github/workflows/ci.yml
```

2. **Add CI Configuration**
```yaml
name: Node.js CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests
      run: npm test
    
    - name: Build
      run: npm run build
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      if: matrix.node-version == '20.x'
```

3. **Commit and Push**
```bash
git add .github/workflows/ci.yml
git commit -m "Add CI pipeline"
git push
```

4. **Monitor Pipeline**
- Go to repository on GitHub
- Click "Actions" tab
- Watch workflow run
- Review logs for each step

### Lab 3: Multi-Stage Pipeline with Docker

**Objective**: Create pipeline that builds, tests, and pushes Docker image

**Jenkins Pipeline**:
```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.8-openjdk-17'
        }
    }
    
    environment {
        DOCKER_IMAGE = 'myapp'
        DOCKER_TAG = "${env.BUILD_ID}"
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }
        
        stage('Push Docker Image') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-creds') {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push('latest')
                    }
                }
            }
        }
    }
}
```

### Lab 4: Deploy to Kubernetes

**Objective**: Create pipeline that deploys to Kubernetes cluster

**GitHub Actions Workflow**:
```yaml
name: Deploy to Kubernetes

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'
    
    - name: Configure kubeconfig
      run: |
        mkdir -p ~/.kube
        echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > ~/.kube/config
    
    - name: Deploy to Kubernetes
      run: |
        kubectl apply -f k8s/deployment.yaml
        kubectl apply -f k8s/service.yaml
        kubectl rollout status deployment/myapp
    
    - name: Verify deployment
      run: |
        kubectl get pods
        kubectl get services
```

### Lab 5: Blue-Green Deployment Pipeline

**Objective**: Implement zero-downtime deployment strategy

```groovy
pipeline {
    agent any
    
    environment {
        APP_NAME = 'myapp'
        NAMESPACE = 'production'
        GREEN_VERSION = 'v1.0.0'
        BLUE_VERSION = 'v1.0.1'
    }
    
    stages {
        stage('Build') {
            steps {
                sh './build.sh'
            }
        }
        
        stage('Deploy Blue') {
            steps {
                script {
                    sh """
                    kubectl apply -f k8s/blue-deployment.yaml -n ${NAMESPACE}
                    kubectl set image deployment/${APP_NAME}-blue \
                      ${APP_NAME}=${APP_NAME}:${BLUE_VERSION} -n ${NAMESPACE}
                    kubectl rollout status deployment/${APP_NAME}-blue -n ${NAMESPACE}
                    """
                }
            }
        }
        
        stage('Health Check Blue') {
            steps {
                script {
                    def healthy = sh(
                        script: './health-check.sh blue',
                        returnStatus: true
                    ) == 0
                    
                    if (!healthy) {
                        error 'Blue deployment health check failed'
                    }
                }
            }
        }
        
        stage('Switch Traffic') {
            steps {
                input message: 'Switch traffic to Blue deployment?', ok: 'Switch'
                script {
                    sh """
                    kubectl patch service ${APP_NAME} -n ${NAMESPACE} \
                      -p '{\"spec\":{\"selector\":{\"version\":\"blue\"}}}'
                    """
                }
            }
        }
        
        stage('Cleanup Green') {
            steps {
                script {
                    sh """
                    kubectl delete deployment ${APP_NAME}-green -n ${NAMESPACE} || true
                    """
                }
            }
        }
    }
    
    post {
        failure {
            script {
                echo 'Rolling back to previous version'
                sh "kubectl rollout undo deployment/${APP_NAME} -n ${NAMESPACE}"
            }
        }
    }
}
```

---

## Troubleshooting Guide

### Common Jenkins Issues

#### Issue 1: Pipeline Hangs Indefinitely
**Symptoms**: Pipeline stuck on a stage
**Solutions**:
```groovy
// Add timeout
options {
    timeout(time: 1, unit: 'HOURS')
}

// Add stage-specific timeout
stage('Long Running Task') {
    options {
        timeout(time: 30, unit: 'MINUTES')
    }
    steps {
        // ...
    }
}
```

#### Issue 2: Out of Disk Space
**Symptoms**: Build fails with "No space left on device"
**Solutions**:
```groovy
post {
    always {
        cleanWs()
    }
}

// Or configure workspace cleanup
properties {
    buildDiscarder(logRotator(numToKeepStr: '10'))
}
```

#### Issue 3: Credential Issues
**Symptoms**: Authentication failures
**Solutions**:
```groovy
// Properly bind credentials
withCredentials([usernamePassword(
    credentialsId: 'docker-hub',
    usernameVariable: 'DOCKER_USER',
    passwordVariable: 'DOCKER_PASS'
)]) {
    sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS'
}
```

### Common GitHub Actions Issues

#### Issue 1: Workflow Not Triggering
**Symptoms**: Push events don't start workflow
**Solutions**:
- Check workflow file location: `.github/workflows/`
- Verify file extension: `.yml` or `.yaml`
- Check branch filters in `on:` section
- Ensure workflow file has correct syntax

#### Issue 2: Secrets Not Available
**Symptoms**: Empty secret values
**Solutions**:
```yaml
# Verify secret is set in repository settings
# Use correct syntax
env:
  MY_SECRET: ${{ secrets.MY_SECRET }}

# For forked repos, secrets aren't shared by default
```

#### Issue 3: Job Timeout
**Symptoms**: Job cancelled after 6 hours
**Solutions**:
```yaml
jobs:
  long-running-job:
    runs-on: ubuntu-latest
    timeout-minutes: 360  # Set appropriate timeout
    
    steps:
      # Optimize long-running steps
      # Consider breaking into multiple jobs
```

### Debugging Techniques

#### Jenkins Debugging
```groovy
// Enable debug logging
println "Debug: Variable value = ${MY_VAR}"

// Pause for debugging
input message: 'Pause for debugging', ok: 'Continue'

// Capture command output
def output = sh(script: 'my-command', returnStdout: true).trim()
echo "Command output: ${output}"
```

#### GitHub Actions Debugging
```yaml
# Enable step debugging
- name: Debug step
  run: |
    echo "All environment variables:"
    printenv
    echo "Current directory:"
    pwd
    echo "Files:"
    ls -la

# Enable runner diagnostic logging
# Set secret: ACTIONS_RUNNER_DEBUG: true
# Set secret: ACTIONS_STEP_DEBUG: true
```

---

## Knowledge Check

### Quiz Questions

#### Question 1: What's the difference between Continuous Delivery and Continuous Deployment?
<details>
<summary>Click for Answer</summary>

**Answer**: 
- **Continuous Delivery**: Code is automatically deployed to staging, but production deployment requires manual approval
- **Continuous Deployment**: Every change that passes tests is automatically deployed to production without manual intervention
</details>

#### Question 2: Which Jenkins pipeline syntax is recommended for most use cases?
<details>
<summary>Click for Answer</summary>

**Answer**: Declarative Pipeline - it's more structured, easier to read, and has better error handling than Scripted Pipeline
</details>

#### Question 3: How do you securely handle secrets in GitHub Actions?
<details>
<summary>Click for Answer</summary>

**Answer**: 
```yaml
env:
  API_KEY: ${{ secrets.API_KEY }}
```
Never hardcode secrets in workflow files. Store them in repository/organization secrets.
</details>

#### Question 4: What is the purpose of the `needs` keyword in GitHub Actions?
<details>
<summary>Click for Answer</summary>

**Answer**: The `needs` keyword defines job dependencies, ensuring jobs run in the correct order. A job with `needs` will only run after the specified jobs complete successfully.
</details>

#### Question 5: Explain the benefit of Pipeline as Code
<details>
<summary>Click for Answer</summary>

**Answer**: 
- Version control for pipelines
- Code review process
- Consistency across environments
- Easy disaster recovery
- Documentation through code
- Reusability and templating
</details>

#### Question 6: What strategy would you use for zero-downtime deployments?
<details>
<summary>Click for Answer</summary>

**Answer**: Blue-Green Deployment or Canary Deployment
- **Blue-Green**: Two identical environments, switch traffic instantly
- **Canary**: Gradually route traffic to new version
- **Rolling Update**: Update instances one by one
</details>

#### Question 7: How can you speed up CI/CD pipelines?
<details>
<summary>Click for Answer</summary>

**Answer**: 
- Run tests in parallel
- Implement caching for dependencies
- Use incremental builds
- Optimize test suites
- Use faster runners
- Split monolithic pipelines
</details>

### Practical Exercises

#### Exercise 1: Create a Multi-Branch Pipeline
Create a Jenkins pipeline that:
- Builds on every commit to any branch
- Runs different tests based on branch name
- Deploys to staging on `develop` branch
- Deploys to production on `main` branch

#### Exercise 2: Implement Approval Gate
Create a GitHub Actions workflow that:
- Runs tests on pull requests
- Requires manual approval before deploying to production
- Sends Slack notification on deployment

#### Exercise 3: Build Matrix Strategy
Create a pipeline that tests your application on:
- 3 different Node.js versions
- 2 different operating systems
- Both with and without optional dependencies

#### Exercise 4: Implement Rollback Mechanism
Create a pipeline that:
- Deploys new version
- Runs health checks
- Automatically rolls back if health checks fail
- Notifies team of rollback

---

## Additional Resources

### Books
- "Continuous Delivery" by Jez Humble and David Farley
- "Jenkins 2: Up and Running" by Brent Laster
- "Learning GitHub Actions" by Brent Laster

### Online Courses
- Jenkins Official Documentation: https://www.jenkins.io/doc/
- GitHub Actions Docs: https://docs.github.com/en/actions
- Udemy: "Jenkins From Zero To Hero"
- Coursera: "Continuous Integration/Continuous Deployment"

### Tools & Plugins
- **Jenkins**: Blue Ocean, Pipeline, Docker, Kubernetes plugins
- **GitHub Actions**: Marketplace actions
- **Testing**: Jest, pytest, JUnit
- **Security**: Snyk, SonarQube, Trivy
- **Monitoring**: Prometheus, Grafana

### Communities
- Jenkins Community: https://www.jenkins.io/community/
- GitHub Community Forum
- DevOps subreddit: r/devops
- Stack Overflow: [jenkins], [github-actions] tags

---

## Next Steps

Now that you've mastered CI/CD:

1. ✅ Complete all hands-on labs
2. ✅ Build a real project with full CI/CD pipeline
3. ✅ Experiment with different deployment strategies
4. ✅ Move to **Module 05: Containerization with Docker**

### Project Challenge
Create a complete CI/CD pipeline for a sample application that:
- Builds on every commit
- Runs comprehensive test suite
- Performs security scanning
- Builds and pushes Docker image
- Deploys to staging automatically
- Requires approval for production
- Implements rollback capability
- Sends notifications on success/failure

**Good luck! 🚀**
