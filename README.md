# CI/CD Pipeline - Task Manager

## 📋 Overview

This project uses a complete **CI/CD (Continuous Integration and Continuous Deployment)** pipeline implemented with **GitHub Actions**. The pipeline automates code quality validation, testing, security scanning, Docker image creation, integration testing, performance testing, and deployments to staging and production environments.

> **Important:** The purpose of this repository is to demonstrate a robust CI/CD workflow. The frontend and backend implementations are not the primary focus of the project and therefore are intentionally not documented in detail. They exist only to support and validate the CI/CD process.

---

# 🎯 Pipeline Objectives

The pipeline was designed to ensure:

- Consistent code quality standards.
- Automated unit testing.
- Dependency and source code security validation.
- Automated Docker image creation and publication.
- Integration testing against a running application.
- Basic performance and load validation.
- Automated deployments to staging and production environments.
- Deployment status notifications.

---

# 🚀 Workflow Triggers

The workflow runs under the following conditions:

## Push Events

```yaml
push:
  branches: [main, develop]
```

Triggered whenever code is pushed to:

- `main`
- `develop`

## Pull Requests

```yaml
pull_request:
  branches: [main]
```

Triggered when a Pull Request targets the `main` branch.

## Manual Execution

```yaml
workflow_dispatch:
```

Allows manual execution directly from the GitHub Actions interface.

---

# 🔧 Global Environment Variables

The workflow defines shared variables used across multiple jobs:

```yaml
env:
  PYTHON_VERSION: '3.11'
  NODE_VERSION: '24'
```

| Variable | Description |
|-----------|-------------|
| `PYTHON_VERSION` | Default Python version used in the workflow |
| `NODE_VERSION` | Default Node.js version |

---

# 📐 Pipeline Flow

The following diagram illustrates the complete CI/CD workflow:
<p align="center">
<img src="docs/images/diagrama CI_CD.jpeg"
</p>

<p align="center">
<em>Figure 1 - End-to-end CI/CD workflow implemented with GitHub Actions.</em>
</p>


---

# 🔍 Stage 1 - Lint and Code Quality

## Job

```yaml
lint
```

## Purpose

Ensure that the source code follows formatting, style, and quality standards before any testing phase begins.

## Main Activities

### Checkout Source Code

```yaml
actions/checkout@v4
```

Downloads the latest version of the repository.

### Configure Python

```yaml
actions/setup-python@v5
```

Installs the configured Python version.

### Dependency Caching

```yaml
actions/cache@v4
```

Caches Python dependencies to speed up future workflow executions.

### Install Tools

```bash
pip install flake8 black isort
```

Tools used:

- **Black** – Code formatting validation
- **isort** – Import organization validation
- **Flake8** – Static analysis and linting

---

## Code Formatting Validation

### Black

```bash
black --check --diff .
```

Verifies that all files comply with formatting standards.

### isort

```bash
isort . --profile black --check-only --diff
```

Ensures imports are organized consistently.

### Flake8

```bash
flake8 . --count --select=E9,F63,F7,F82
```

Detects critical syntax and linting errors.

---

# ✅ Stage 2 - Unit Tests

## Job

```yaml
test
```

## Dependency

```yaml
needs: lint
```

Unit tests execute only after code quality checks pass.

---

## Multi-Version Testing

Tests are executed against multiple Python versions:

```text
Python 3.9
Python 3.10
Python 3.11
```

This guarantees compatibility across supported runtime environments.

---

## Test Execution

```bash
pytest --verbose \
       --tb=short \
       --cov=. \
       --cov-report=xml \
       --cov-report=term
```

### Validation Scope

- Business logic correctness
- Regression detection
- Test coverage measurement

---

## Coverage Reporting

Coverage reports generated on Python 3.11 are uploaded to Codecov.

```yaml
codecov/codecov-action@v4
```

This provides centralized visibility into test coverage metrics.

---

# 🔐 Stage 3 - Security Scan

## Job

```yaml
security
```

## Dependency

```yaml
needs: lint
```

Runs after successful code quality validation.

---

## Dependency Vulnerability Analysis

### Safety

```bash
safety check
```

Checks installed Python packages against known vulnerability databases.

---

## Static Security Analysis

### Bandit

```bash
bandit -r .
```

Analyzes Python source code for common security issues and insecure coding patterns.

---

## Security Report

Results are exported to:

```text
bandit-report.json
```

The report is uploaded as a workflow artifact and remains available for later review.

---

# 🐳 Stage 4 - Docker Image Build

## Job

```yaml
build
```

## Dependencies

```yaml
needs:
  - test
  - security
```

Docker image generation begins only after all tests and security checks pass.

---

## Docker Hub Authentication

Executed only when the workflow is not running as a Pull Request.

```yaml
docker/login-action@v3
```

Uses the following GitHub Secrets:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`

---

## Automatic Image Tagging

The pipeline generates tags based on:

- Branch names
- Pull Requests
- Commit SHA
- Latest release

Examples:

```text
task-manager:develop
task-manager:main
task-manager:sha-abc123
task-manager:latest
```

---

## Multi-Architecture Build

The image is built for:

```text
linux/amd64
linux/arm64
```

This allows compatibility with different deployment environments.

---

## Build Cache Optimization

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

Build cache significantly reduces image build times in future runs.

---

# 🔗 Stage 5 - Integration Tests

## Job

```yaml
Integration-test
```

## Dependency

```yaml
needs: build
```

## Condition

This stage is skipped for Pull Requests.

```yaml
if: github.event_name != 'pull_request'
```

---

## Purpose

Validate the application behavior after packaging it into a Docker container.

---

## Workflow

### Build Local Test Image

```bash
docker build -t local-task-manager:test .
```

### Start Container

```bash
docker run -d -p 8000:8000
```

### Check Application Health

```http
GET /api/health
```

The pipeline waits until the application is fully available before proceeding.

---

## API Validation

The integration tests perform a complete CRUD cycle.

### Create Task

```http
POST /api/tasks
```

### Retrieve Task

```http
GET /api/tasks/{id}
```

### Update Task

```http
PUT /api/tasks/{id}
```

### Delete Task

```http
DELETE /api/tasks/{id}
```

### Get Statistics

```http
GET /api/stats
```

---

## Frontend Availability Validation

Although the frontend is not part of the project's focus, the pipeline confirms that the application's static resources are accessible.

### Checks Performed

```http
GET /
GET /static/style.css
GET /static/script.js
```

These checks only verify availability and successful delivery of assets.

---

# ⚡ Stage 6 - Performance Testing

## Job

```yaml
performance-test
```

## Dependency

```yaml
needs: integration-test
```

---

## Tool

```text
Locust
```

Locust is used to execute basic load and performance testing scenarios.

---

## Test Configuration

```text
Users: 100
Spawn Rate: 10 users/second
Duration: 2 minutes
```

Command executed:

```bash
locust \
  --users 100 \
  --spawn-rate 10 \
  --run-time 2m \
  --headless
```

---

## Purpose

Measure:

- Responsiveness under load
- Stability during concurrent access
- Basic performance characteristics

These tests provide an additional quality gate before deployment.

---

# 🚀 Stage 7 - Deploy to Staging

## Job

```yaml
deploy-staging
```

## Dependencies

```yaml
needs:
  - integration-test
  - performance-test
```

## Execution Condition

Runs only when code is pushed to:

```text
develop
```

---

## Environment

```yaml
environment: staging
```

URL:

```text
https://task-manager-staging.example.com
```

---

## Purpose

Automatically deploy validated builds to a staging environment for further verification and acceptance testing.

---

# 🌐 Stage 8 - Deploy to Production

## Job

```yaml
deploy-production
```

## Dependencies

```yaml
needs:
  - integration-test
  - performance-test
```

## Execution Condition

Runs only when code is pushed to:

```text
main
```

---

## Environment

```yaml
environment: production
```

URL:

```text
https://task-manager.example.com
```

---

## Purpose

Automatically release approved versions to the production environment.

---

# 📢 Stage 9 - Deployment Notification

## Job

```yaml
notify
```

## Dependencies

```yaml
needs:
  - deploy-staging
  - deploy-production
```

---

## Purpose

Provide visibility regarding deployment results.

### Example Output

```text
Deployment successful!
Staging: success
Production: success
```

Notifications only run when at least one deployment stage has been executed.

---

# 🔑 Required GitHub Secrets

The following repository secrets must be configured:

| Secret | Purpose |
|----------|----------|
| `DOCKERHUB_USERNAME` | Docker Hub account username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |

---

# 📊 Benefits of This Pipeline

This implementation provides:

- Automated code quality validation
- Multi-version Python compatibility testing
- Continuous security analysis
- Automated Docker image creation
- Integration testing against containerized builds
- Performance validation before deployment
- Automated environment promotion
- End-to-end deployment automation
- Traceability through image tags and commit SHAs
- Reduced manual intervention and deployment risk

---

# 📝 Final Notes

This workflow was designed to showcase modern CI/CD and DevOps practices, emphasizing automation, reliability, quality assurance, and deployment consistency.

The frontend and backend implementations are intentionally outside the scope of this documentation. Their role is only to provide a functional application that can be used to demonstrate the CI/CD lifecycle. Therefore, architectural details, business logic, and implementation specifics of those components are not covered here.

The primary focus of this repository is the CI/CD pipeline itself and the processes that support automated software delivery.