# Piksel Project Repository and CI/CD Strategy

## Overview

This document outlines the proposed multi-repository architecture for the Piksel Earth Observation Data platform. The strategy balances separation of concerns with system-wide coordination to enable efficient development and deployment of all components.

## Repository Structure

```mermaid
graph LR
    org[Piksel Organization] --> infra[piksel-infra]
    org --> core[piksel-core]
    org --> jupyter[piksel-jupyter]
    org --> k8s[piksel-kubernetes]
    org --> doc[piksel-document]
    org --> test[piksel-tests]

    infra --> terraform[Terraform IaC]
    terraform --> main[Root Configuration]
    terraform --> modules[Modules]

    k8s --> eks[EKS Configuration]
    k8s --> flux[Flux GitOps]
    k8s --> helm[Helm Charts]

    core --> docker_core[Container Definitions]
    core --> products[Product Definitions]

    jupyter --> docker_jupyter[Jupyter/Dask Setup]
    jupyter --> notebooks[Analysis Examples]

    doc --> system_docs[System Documentation]
    doc --> api_docs[API Documentation]

    test --> integration[Integration Tests]
    test --> validation[Deployment Validation]
```

## Repository Details

### 1. `piksel-infra`

Infrastructure as Code repository containing all AWS resource definitions.

<!-- prettier-ignore-start -->
```markdown
piksel-infra/
├── .github/
│   └── workflows/
│       └── tf-validate.yml     # Only format and validation checks
├── .gitignore
├── .pre-commit-config.yaml
├── README.md
├── modules/                    # Only for complex/unique configurations
│   └── custom-setup/           # Example: Special configuration needed
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── shared/                     # For shared resources like ECR and Route53
│   ├── README.md               
│   ├── main.tf                 # Defines the piksel-core ECR repository
│   ├── variables.tf            # Variables for shared resources
│   ├── outputs.tf              # Outputs for shared resources 
│   ├── providers.tf            # AWS Provider configuration for shared resources
│   └── shared.auto.tfvars      # Environment Specific Variable Declarations
├── dev/                        # Dev environment specifics
│   ├── README.md              
│   ├── main.tf                 # Uses registry modules
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   └── dev.auto.tfvars
├── staging/                    # Staging environment specifics
│   └── [same structure as dev]
└── prod/                       # Production environment specifics
    └── [same structure as dev]
```
<!-- prettier-ignore-end -->

### 2. `piksel-core`

Open Data Cube core services repository.

<!-- prettier-ignore-start -->
```markdown
piksel-core/
├── docker/                     # Container definitions for ODC
├── products/                   # Earth observation product definitions
└── .github/workflows/          # Core services CI/CD pipelines
```
<!-- prettier-ignore-end -->

### 3. `piksel-jupyter`

Custom Jupyter environment with Dask integration.

<!-- prettier-ignore-start -->
```markdown
piksel-jupyter/
├── config/                     # JupyterHub/Dask config
├── notebooks/                  # Example and template notebooks
└── .github/workflows/          # Jupyter image CI/CD pipelines
```
<!-- prettier-ignore-end -->

### 4. `piksel-kubernetes`

Kubernetes and GitOps management:

<!-- prettier-ignore-start -->
```markdown
piksel-kubernetes/
├── terraform/         # EKS infrastructure
├── helm-charts/       # Application Helm charts
│   ├── core/          # Core service charts
│   ├── addons/        # Cluster add-on charts
│   └── values/        # Environment-specific values
├── flux/              # Flux GitOps configurations
└── docs/              # Kubernetes-specific documentation
```
<!-- prettier-ignore-end -->

### 5. `piksel-website`

Landing page and documentation site.

<!-- prettier-ignore-start -->
```markdown
piksel-website/
├── public/                     # Static website content
└── .github/workflows/          # Website deployment workflows
```
<!-- prettier-ignore-end -->

### 6. `piksel-tests`

Testing and validation:

<!-- prettier-ignore-start -->
```markdown
piksel-tests/           # [Rename from piksel-test]
├── integration-tests/  # Cross-service test suites
├── performance-tests/ # Load testing and benchmarks
├── environments/      # Test environment configurations
└── docs/             # Testing documentation
```
<!-- prettier-ignore-end -->

### 7. `piksel-document`

Centralized documentation:

<!-- prettier-ignore-start -->
```markdown
piksel-document/
├── security/           # Security foundations, policies, compliance
├── project-plans/      # Project plans, milestones 
├── achitecture/        # System architecture
├── operations/         # Runbooks, procedures, how-tos
├── assets/             # Logos, and etc
└── api/                # API documentation
```
<!-- prettier-ignore-end -->

## Branching Strategy

### Infrastructure Repositories (piksel-infra, piksel-kubernetes)

```mermaid
graph TD
    M[main] --> F1[feature/xyz]
    M --> F2[feature/abc]
    M --> H[hotfix/fix-123]

    F1 --> PR1[Pull Request]
    F2 --> PR2[Pull Request]
    H --> PR3[Pull Request]

    PR1 --> M
    PR2 --> M
    PR3 --> M
```

**Branch Protection Rules**

- **Main Branch** (`main`)

  - Require a pull request before merging
    - Required number of approvals: 1
    - Dismiss stale pull request approvals when new commits are pushed
  - Require status checks to pass before merging
    - Require branches to be up to date before merging
    - Required status checks: build, test
  - Do not allow bypassing the above settings

- **Feature Branches** (`feature/*`)
  - Standard pull request workflow
  - No specific protection rules

### Workflow Guidelines for Infrastructure Repo (piksel-infra, piksel-kubernetes)

1. **Feature Development**

   - Create feature branch from main
   - Submit pull request when ready
   - Get approval from 1 reviewer
   - Ensure format/validation checks pass
   - Merge to main

2. **Release Process**

   - Infrastructure: Update appropriate environment directory
   - Applications: Create new tag (v1.0.0, v1.1.0, etc.)

3. **Hotfix Process**
   - Create hotfix branch from main
   - Follow same PR process as features
   - Automated deployment after merge via respective tools
     - Terraform Cloud for infrastructure
     - FluxCD for Kubernetes resources

### Application Repositories (piksel-core, piksel-jupyter)

```mermaid
graph TD
    M[main] --> F1[feature/new-api]
    M --> F2[feature/ui-update]
    M --> T[tags/v1.0.0]

    F1 --> PR1[Pull Request]
    F2 --> PR2[Pull Request]

    PR1 --> M
    PR2 --> M
```

**Branch Protection Rules**

- **Main Branch**

  - Require a pull request before merging
    - Required number of approvals: 1
    - Dismiss stale pull request approvals when new commits are pushed
  - Require status checks to pass before merging
    - Require branches to be up to date before merging
    - Required status checks: build, test
  - Do not allow bypassing the above settings

- **Feature Branches** (`feature/*`)
  - Standard pull request workflow
  - No specific protection rules

## CI/CD Strategy

### 1. Infrastructure Repository CI/CD (piksel-infra)

```mermaid
flowchart LR
    A[Push to feature/*] --> B[Terraform Format Check]
    B --> C[Terraform Validate]
    C --> D[TFSec Scan]
    D --> E[PR Review]

    F[Merge to main] --> G[Terraform Cloud Trigger]
    G --> H[TF Cloud Plan Dev]
    H --> I[TF Cloud Apply Dev]
    I --> J[TF Cloud Plan Staging]
    J --> K[TF Cloud Apply Staging]
    K --> L[TF Cloud Plan Prod]
    L --> M[TF Cloud Apply Prod]
```

**GitHub Actions Workflow Structure**

```yaml
name: Terraform Validate
on:
  pull_request:
    branches: [main]
    paths:
      - "**/*.tf"
      - "**/*.tfvars"

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: true # Won't fail the pipeline but will show warnings

      - name: Terraform Init & Validate
        run: |
          cd dev && terraform init -backend=false && terraform validate
          cd ../staging && terraform init -backend=false && terraform validate
          cd ../prod && terraform init -backend=false && terraform validate
```

**Key Components**

- **Authentication:**
  - Terraform Cloud manages AWS authentication via OIDC
  - GitHub Actions authenticates with Terraform Cloud via API tokens
- **State Management**:
  - Separate Terraform Cloud workspaces per environment
  - State locked and managed in Terraform Cloud
- **Security:**
  - Pre-commit hooks for local security checks
  - TFSec scans in PR checks for IaC security validation
  - TFLint for Terraform best practices enforcement
- **Environment Protection**:
  - Required approvals in Terraform Cloud for staging/prod
  - Manual approval gates between environments

**Monitoring Strategy**

- **Key metrics monitored:**

  - EKS cluster health
    - Node CPU/Memory utilization
    - Pod status
    - Control plane metrics
  - RDS metrics
    - Database connections
    - Query performance
    - Storage capacity
  - Infrastructure events
    - Deployment status
    - Security group changes
    - Network anomalies

- **Notification channels**:
  - Critical alerts: PagerDuty
  - Warning alerts: Slack
  - Status updates: Email

### 2. Application Repositories CI/CD (piksel-core)

```mermaid
flowchart LR
    A[Push to feature/*] --> B[Build & Test]
    B --> C[Container Build]
    C --> D[Security Scan]
    D --> E[PR Review]

    F[Merge to main] --> G[Build & Test]
    G --> H[Container Build]
    H --> I[Security Scan]
    I --> J[Push to ECR :latest]
    J --> K[Deploy to Dev]

    L[Create Tag] --> M[Container Build]
    M --> N[Push to ECR :v1.x.x]
    N --> O[Update Helm Values]
```

**GitHub Actions Workflow Structure**

```yaml
# Feature Branch Workflow
name: Feature Branch Checks
on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - unit-tests
      - container-build
      - trivy-scan

# Main Branch Workflow
name: Main Deployment
on:
  push:
    branches: [main]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - build-container
      - push-to-ecr
      - deploy-to-dev

# Release Workflow
name: Release
on:
  push:
    tags: ['v*']

jobs:
  release:
    steps:
      - build-container
      - push-to-ecr
      - update-helm-values
```

**Key Components**

- **Authentication**:
  - GitHub Actions OIDC with AWS for ECR pushes
  - Limited IAM roles for ECR operations
- **Container Registry**:
  - ECR with image scanning enabled
  - Tag immutability
- **Deployment**:

  - FluxCD for GitOps-based deployments
  - Helm charts in piksel-kubernetes repo

I'll address each point and provide updated CI/CD strategies:

### 3. Jupyter Repository CI/CD (piksel-jupyter)

```mermaid
flowchart LR
    A[Push to feature/*] --> B[Config Validation]
    B --> C[Notebook Linting]
    C --> D[PR Review]

    E[Merge to main] --> F[Config Validation]
    F --> G[Update Config Maps]
    G --> H[Sync Notebooks to S3]
```

**GitHub Actions Workflow**

```yaml
name: Jupyter Config and Notebook Management
on:
  push:
    branches: [main]
    paths:
      - "config/**"
      - "notebooks/**"

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - validate-jupyter-config
      - lint-notebooks

  deploy:
    needs: validate
    steps:
      - name: Update ConfigMaps
        if: contains(github.event.paths, 'config/')
        run: |
          # Update k8s configmaps via flux

      - name: Sync Notebooks to S3
        if: contains(github.event.paths, 'notebooks/')
        run: |
          aws s3 sync notebooks/ s3://piksel-notebooks/
```

**Key Components**:

- Focuses on config validation and notebook management
- Uses Flux GitOps for config changes
- S3 for notebook storage with versioning

### 4. Website CI/CD (piksel-website)

```mermaid
flowchart LR
    A[Push to main] --> B[Build Static Site]
    B --> C[Run Tests]
    C --> D[Deploy to S3]
    D --> E[Invalidate CloudFront]
```

**GitHub Actions Workflow**

```yaml
name: Website Deployment
on:
  push:
    branches: [main]
    paths: ["public/**"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Build site
        run: # Static site build commands

      - name: Deploy to S3
        run: |
          aws s3 sync public/ s3://piksel-website/

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation --distribution-id ${{ secrets.CF_DIST_ID }}
```

**Key Components**:

- Simple S3 + CloudFront deployment
- CloudFront invalidation for immediate updates

### 5. Test Repository CI/CD (piksel-tests)

```mermaid
flowchart LR
    A[Push to feature/*] --> B[Lint Tests]
    B --> C[Unit Tests]
    C --> D[PR Review]

    E[Merge to main] --> F[Run Integration Tests]
    F --> G[Run Performance Tests]
    G --> H[Generate Test Reports]
    H --> I[Update Test Dashboard]
```

**GitHub Actions Workflow**

```yaml
# Feature Branch Workflow
name: Test Validation
on:
  pull_request:
    branches: [main]
    paths:
      - 'integration-tests/**'
      - 'performance-tests/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - test-lint
      - unit-tests
      - generate-report

# Main Branch Workflow
name: Integration Testing
on:
  push:
    branches: [main]

jobs:
  integration:
    environment: test
    steps:
      - integration-tests
      - performance-tests
      - publish-report

```

I'll update the CI/CD Strategy section following the original document's structure:

## CI/CD Strategy

### 2. Kubernetes Repository CI/CD (piksel-kubernetes)

```mermaid
flowchart LR
    A[Push to feature/*] --> B[Helm Lint]
    B --> C[Kubeval Check]
    C --> D[PR Review]

    E[Merge to main] --> F[FluxCD Detection]
    F --> G[Sync Dev]
    G --> H[Integration Tests]
    H --> I[Sync Staging]
    I --> J[Sync Production]
```

**GitHub Actions Workflow Structure**

```yaml
# Feature Branch Workflow
name: Kubernetes Validate
on:
  pull_request:
    branches: [main]
    paths:
      - "helm-charts/**"
      - "flux/**"

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Helm
        uses: azure/setup-helm@v3

      - name: Run Helm lint
        run: |
          for chart in helm-charts/*; do
            if [ -d "$chart" ]; then
              helm lint $chart
            fi
          done

      - name: Setup Kubeval
        run: |
          wget https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz
          tar xf kubeval-linux-amd64.tar.gz
          sudo cp kubeval /usr/local/bin

      - name: Validate Kubernetes manifests
        run: |
          kubeval --strict flux/**/*.yaml
```

### Branch Strategy

[This section remains unchanged as it's still valid for both repositories]

### Workflow Guidelines

1. **Feature Development**

   - Create feature branch from main
   - Submit pull request when ready
   - Get approval from 1 reviewer
   - Ensure format/validation checks pass
   - Merge to main

2. **Release Process**

   - piksel-infra: Terraform Cloud automatically applies changes
   - piksel-kubernetes: FluxCD automatically syncs changes

3. **Hotfix Process**
   - Create hotfix branch from main
   - Follow same PR process as features
   - Automated deployment after merge via respective tools
     - Terraform Cloud for infrastructure
     - FluxCD for Kubernetes resources

Would you like me to elaborate on any specific section?
