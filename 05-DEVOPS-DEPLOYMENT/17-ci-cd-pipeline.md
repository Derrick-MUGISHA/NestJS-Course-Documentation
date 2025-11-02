# Module 17: CI/CD Pipeline

## 📚 Table of Contents
1. [Overview](#overview)
2. [What is CI/CD?](#what-is-cicd)
3. [GitHub Actions Setup](#github-actions-setup)
4. [CI Pipeline Configuration](#ci-pipeline-configuration)
5. [CD Pipeline Configuration](#cd-pipeline-configuration)
6. [Docker in CI/CD](#docker-in-cicd)
7. [Kubernetes Deployment](#kubernetes-deployment)
8. [Secrets Management](#secrets-management)
9. [Multi-Environment Deployment](#multi-environment-deployment)
10. [Best Practices](#best-practices)
11. [Mini Project: Complete CI/CD](#mini-project-complete-cicd)

---

## Overview

CI/CD (Continuous Integration/Continuous Deployment) automates building, testing, and deploying applications. This module covers setting up complete CI/CD pipelines using GitHub Actions.

**CI/CD Benefits:**
- **Automated Testing**: Run tests on every commit
- **Fast Feedback**: Catch issues early
- **Consistent Deployments**: Same process every time
- **Reduced Errors**: Less manual intervention
- **Faster Releases**: Deploy multiple times per day

**What You'll Learn:**
- Set up GitHub Actions workflows
- Automate testing and building
- Build and push Docker images
- Deploy to Kubernetes
- Manage secrets securely

---

## What is CI/CD?

### Continuous Integration (CI)

**CI** automates building and testing code whenever changes are pushed to the repository.

**CI Process:**
1. Developer pushes code
2. CI server detects changes
3. Runs automated tests
4. Builds the application
5. Reports results

### Continuous Deployment (CD)

**CD** automatically deploys code changes to production after passing tests.

**CD Process:**
1. Code passes CI tests
2. Build production artifacts
3. Deploy to staging
4. Run smoke tests
5. Deploy to production

---

## GitHub Actions Setup

GitHub Actions is built into GitHub and provides CI/CD capabilities.

### Workflow Files

Workflows are defined in YAML files in `.github/workflows/` directory:

```
.github/
└── workflows/
    ├── ci.yml          # Continuous Integration
    ├── cd.yml          # Continuous Deployment
    └── release.yml     # Release workflow
```

### Basic Workflow Structure

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

# When to trigger the workflow
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

# Jobs to run
jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Run tests
      run: npm test
    
    - name: Run E2E tests
      run: npm run test:e2e
```

**Workflow Components:**
- `name`: Workflow name
- `on`: Trigger events
- `jobs`: Work to be done
- `steps`: Individual tasks

---

## CI Pipeline Configuration

### Complete CI Workflow

```yaml
# .github/workflows/ci.yml
name: Continuous Integration

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

# Concurrency: Cancel in-progress runs for same branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # Job 1: Lint and Type Check
  lint:
    name: Lint and Type Check
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run ESLint
      run: npm run lint
    
    - name: Check TypeScript
      run: npx tsc --noEmit

  # Job 2: Unit Tests
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run unit tests
      run: npm run test:cov
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage/lcov.info

  # Job 3: E2E Tests
  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run migrations
      run: npm run migration:run
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
    
    - name: Run E2E tests
      run: npm run test:e2e
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
        REDIS_HOST: localhost
        REDIS_PORT: 6379

  # Job 4: Build
  build:
    name: Build Application
    runs-on: ubuntu-latest
    needs: [lint, unit-tests, e2e-tests]  # Run after tests pass
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Build application
      run: npm run build
    
    - name: Upload build artifacts
      uses: actions/upload-artifact@v3
      with:
        name: dist
        path: dist/
```

**CI Pipeline Stages:**
1. **Lint**: Code quality checks
2. **Unit Tests**: Fast unit tests
3. **E2E Tests**: Integration tests with services
4. **Build**: Compile application
5. **Artifacts**: Save build output

---

## CD Pipeline Configuration

### Deployment Workflow

```yaml
# .github/workflows/cd.yml
name: Continuous Deployment

on:
  push:
    branches: [ main ]
  workflow_dispatch:  # Manual trigger

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Build and push Docker image
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  # Deploy to Kubernetes
  deploy:
    name: Deploy to Kubernetes
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set up kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig
    
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/nestjs-app \
          nestjs=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
          -n production
        
        kubectl rollout status deployment/nestjs-app -n production
```

---

## Docker in CI/CD

### Docker Build with Caching

```yaml
- name: Build Docker image
  uses: docker/build-push-action@v4
  with:
    context: .
    push: true
    tags: my-app:latest
    cache-from: type=registry,ref=my-app:buildcache
    cache-to: type=registry,ref=my-app:buildcache,mode=max
```

### Multi-platform Build

```yaml
- name: Set up QEMU
  uses: docker/setup-qemu-action@v2

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v2

- name: Build for multiple platforms
  uses: docker/build-push-action@v4
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: my-app:latest
```

---

## Kubernetes Deployment

### Deploy with kubectl

```yaml
- name: Deploy to Kubernetes
  run: |
    # Apply Kubernetes manifests
    kubectl apply -f k8s/
    
    # Set image
    kubectl set image deployment/nestjs-app \
      nestjs=my-app:${{ github.sha }}
    
    # Wait for rollout
    kubectl rollout status deployment/nestjs-app
```

### Using Helm

```yaml
- name: Set up Helm
  uses: azure/setup-helm@v3
  with:
    version: '3.12.0'

- name: Deploy with Helm
  run: |
    helm upgrade --install nestjs-app ./helm-chart \
      --set image.tag=${{ github.sha }} \
      --namespace production
```

---

## Secrets Management

### GitHub Secrets

Store secrets in GitHub repository settings:

1. Go to Settings → Secrets and variables → Actions
2. Add secrets: `DATABASE_PASSWORD`, `JWT_SECRET`, etc.

### Using Secrets in Workflows

```yaml
- name: Set environment variables
  run: |
    echo "DATABASE_PASSWORD=${{ secrets.DATABASE_PASSWORD }}" >> $GITHUB_ENV

- name: Deploy
  env:
    JWT_SECRET: ${{ secrets.JWT_SECRET }}
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: npm run deploy
```

### Kubernetes Secrets

```yaml
- name: Create Kubernetes secret
  run: |
    kubectl create secret generic app-secrets \
      --from-literal=database-password="${{ secrets.DATABASE_PASSWORD }}" \
      --from-literal=jwt-secret="${{ secrets.JWT_SECRET }}" \
      --dry-run=client -o yaml | kubectl apply -f -
```

---

## Multi-Environment Deployment

### Environment-specific Workflows

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy to Staging

on:
  push:
    branches: [ develop ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
    - name: Deploy to staging
      run: |
        kubectl set image deployment/nestjs-app \
          nestjs=my-app:${{ github.sha }} \
          -n staging
```

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    
    steps:
    - name: Deploy to production
      run: |
        kubectl set image deployment/nestjs-app \
          nestjs=my-app:${{ github.sha }} \
          -n production
```

---

## Best Practices

### 1. Use Matrix Builds

```yaml
strategy:
  matrix:
    node-version: [16, 18, 20]
steps:
- uses: actions/setup-node@v3
  with:
    node-version: ${{ matrix.node-version }}
```

### 2. Cache Dependencies

```yaml
- uses: actions/setup-node@v3
  with:
    cache: 'npm'
```

### 3. Parallel Jobs

Run independent jobs in parallel for faster CI.

### 4. Fail Fast

Stop pipeline on first failure to save time.

### 5. Use Environments

Protect production with manual approval.

### 6. Tag Docker Images

Tag images with commit SHA and version.

---

## Mini Project: Complete CI/CD

Create a complete CI/CD pipeline:

1. **CI Workflow**: Lint, test, build
2. **CD Workflow**: Build Docker, deploy to K8s
3. **Multi-environment**: Staging and production
4. **Secrets**: Secure configuration
5. **Notifications**: Slack/Discord alerts

### Complete Workflow Example

See the examples above for complete implementations.

---

## Next Steps

✅ Set up GitHub Actions  
✅ Automate testing and building  
✅ Deploy to Kubernetes  
✅ Manage secrets securely  
✅ Configure multi-environment deployment  
✅ Move to Module 18: Microservices Architecture  

---

*You now have automated CI/CD pipelines! 🚀*

