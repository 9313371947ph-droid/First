# Module 07: Infrastructure as Code
## Section 07: Terraform Best Practices & CI/CD Integration

### 1. Introduction to Terraform Best Practices

Following best practices ensures your Terraform code is:
- **Maintainable**: Easy to understand and modify
- **Scalable**: Works for small and large deployments
- **Secure**: Follows security principles
- **Reliable**: Predictable and consistent behavior
- **Collaborative**: Enables team productivity

### 2. Code Organization

#### Directory Structure Best Practices

**Recommended Structure:**
```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── compute/
│   │   └── ...
│   └── database/
│       └── ...
├── scripts/
│   ├── validate.sh
│   └── format-check.sh
├── .pre-commit-config.yaml
├── .gitignore
└── README.md
```

**Alternative (Workspace-based):**
```
infrastructure/
├── main.tf
├── variables.tf
├── outputs.tf
├── locals.tf
├── versions.tf
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
├── modules/
│   └── ...
└── .terraform.lock.hcl
```

#### File Naming Conventions

```hcl
# ✅ Good - Clear and consistent
main.tf           # Primary resources
variables.tf      # Input variables
outputs.tf        # Output values
locals.tf         # Local values
versions.tf       # Provider versions
data-sources.tf   # Data sources
resources-ec2.tf  # Specific resource types (for large projects)

# ❌ Bad - Unclear naming
config.tf
stuff.tf
final-final-v2.tf
```

### 3. Variable Management

#### Variable Types and Validation

```hcl
# Use specific types
variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}

variable "allowed_ips" {
  description = "List of allowed IP addresses"
  type        = list(string)
  default     = []
  
  validation {
    condition = alltrue([
      for ip in var.allowed_ips : can(cidrhost(ip, 0))
    ])
    error_message = "All values must be valid CIDR blocks."
  }
}

variable "instance_config" {
  description = "Instance configuration object"
  type = object({
    instance_type = string
    volume_size   = number
    encrypted     = bool
  })
}
```

#### Sensitive Variables

```hcl
# Mark sensitive variables
variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
}

variable "api_key" {
  description = "External API key"
  type        = string
  sensitive   = true
}

# Use secrets management in production
resource "aws_secretsmanager_secret" "db_password" {
  name = "prod/db/password"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = var.db_password
}
```

#### Variable Files

**File: `environments/prod.tfvars`**
```hcl
environment      = "prod"
instance_count   = 5
instance_type    = "t3.xlarge"
enable_monitoring = true
db_password      = "use-secrets-manager"  # Reference, not actual value

tags = {
  Project     = "MyApp"
  CostCenter  = "Engineering"
  Compliance  = "SOC2"
}
```

**Usage:**
```bash
terraform apply -var-file=environments/prod.tfvars
```

### 4. State Management

#### Remote Backend Configuration

```hcl
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    
    # Enable versioning
    # Configure in AWS Console or via Terraform
  }
}

# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  tags = {
    Name = "Terraform State Lock Table"
  }
}
```

#### State Isolation Strategies

**Strategy 1: Separate Buckets per Environment**
```hcl
# dev/backend.tf
backend "s3" {
  bucket = "terraform-state-dev"
  key    = "app/terraform.tfstate"
}

# prod/backend.tf
backend "s3" {
  bucket = "terraform-state-prod"
  key    = "app/terraform.tfstate"
}
```

**Strategy 2: Same Bucket, Different Keys**
```hcl
# dev/backend.tf
backend "s3" {
  bucket = "terraform-state-company"
  key    = "dev/app/terraform.tfstate"
}

# prod/backend.tf
backend "s3" {
  bucket = "terraform-state-company"
  key    = "prod/app/terraform.tfstate"
}
```

**Strategy 3: Workspaces**
```hcl
# Single backend, multiple workspaces
backend "s3" {
  bucket = "terraform-state-company"
  key    = "app/terraform.tfstate"
}

# Workspaces: dev, staging, prod
```

### 5. Security Best Practices

#### Principle of Least Privilege

```hcl
# IAM role for Terraform
resource "aws_iam_role" "terraform" {
  name = "terraform-execution-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

# Minimal permissions
resource "aws_iam_role_policy" "terraform" {
  name = "terraform-policy"
  role = aws_iam_role.terraform.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:Describe*",
          "ec2:RunInstances",
          "ec2:TerminateInstances"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::terraform-state-company/*"
      }
    ]
  })
}
```

#### Never Hardcode Secrets

```hcl
# ❌ BAD - Hardcoded secret
resource "aws_db_instance" "bad" {
  password = "SuperSecret123!"
}

# ✅ GOOD - Use variables with secrets manager
variable "db_password" {
  type      = string
  sensitive = true
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "good" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}

# ✅ BETTER - Auto-generate and store
resource "random_password" "db_password" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db_password.result
}
```

#### Enable Encryption

```hcl
# S3 Bucket encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "bucket" {
  bucket = aws_s3_bucket.data.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform.arn
    }
  }
}

# EBS encryption
resource "aws_ebs_volume" "encrypted" {
  availability_zone = "us-east-1a"
  size              = 100
  encrypted         = true
  kms_key_id        = aws_kms_key.ebs.arn
}

# RDS encryption
resource "aws_db_instance" "encrypted" {
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
}
```

### 6. Testing Terraform Code

#### terraform validate

```bash
# Validate configuration syntax
terraform validate

# In CI/CD
if ! terraform validate; then
  echo "❌ Validation failed"
  exit 1
fi
echo "✅ Validation passed"
```

#### terraform fmt

```bash
# Format files
terraform fmt

# Check formatting (for CI)
terraform fmt -check -recursive

# Auto-format all files
terraform fmt -recursive
```

#### tflint

```bash
# Install tflint
brew install tflint

# Initialize plugins
tflint --init

# Run linting
tflint

# Configuration file: .tflint.hcl
plugin "aws" {
  enabled = true
  deep    = true
  region  = "us-east-1"
}

rule "terraform_deprecated_interpolation" {
  enabled = true
}

rule "terraform_unused_declarations" {
  enabled = true
}
```

#### Terratest (Go-based testing)

```go
// test/terraform_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/stretchr/testify/assert"
)

func TestTerraformWebServer(t *testing.T) {
  terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
    TerraformDir: "../examples/web-server",
    Vars: map[string]interface{}{
      "instance_type": "t3.micro",
    },
  })
  
  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)
  
  instanceId := terraform.Output(t, terraformOptions, "instance_id")
  assert.NotEmpty(t, instanceId)
}
```

### 7. CI/CD Integration

#### GitHub Actions Workflow

**File: `.github/workflows/terraform.yml`**
```yaml
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  TF_VERSION: '1.5.0'
  AWS_REGION: 'us-east-1'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: Terraform Docs
        run: |
          go install github.com/segmentio/terraform-docs@latest
          terraform-docs markdown table . >> README.md
      
      - name: TFLint
        uses: terraform-linters/tflint-action@v2
        with:
          tflint_version: latest

  plan:
    needs: validate
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Plan
        run: terraform plan -out=tfplan -var-file=environments/${{ matrix.environment }}.tfvars
        env:
          TF_VAR_environment: ${{ matrix.environment }}
      
      - name: Upload Plan
        uses: actions/upload-artifact@v3
        with:
          name: tfplan-${{ matrix.environment }}
          path: tfplan

  apply:
    needs: plan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: ${{ matrix.environment }}
    
    strategy:
      matrix:
        environment: [dev, staging, prod]
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Download Plan
        uses: actions/download-artifact@v3
        with:
          name: tfplan-${{ matrix.environment }}
      
      - name: Terraform Init
        run: terraform init
      
      - name: Manual Approval for Prod
        if: matrix.environment == 'prod'
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: admin-user,devops-lead
          minimum-approvals: 2
          issue-title: "Deploy to Production?"
          issue-body: "Please review the Terraform plan and approve deployment"
      
      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
```

#### GitLab CI/CD

**File: `.gitlab-ci.yml`**
```yaml
stages:
  - validate
  - plan
  - apply

variables:
  TF_VERSION: "1.5.0"
  AWS_REGION: "us-east-1"

validate:
  stage: validate
  image: hashicorp/terraform:${TF_VERSION}
  script:
    - terraform init
    - terraform fmt -check -recursive
    - terraform validate
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

plan-dev:
  stage: plan
  image: hashicorp/terraform:${TF_VERSION}
  script:
    - terraform init
    - terraform plan -out=tfplan -var-file=environments/dev.tfvars
  artifacts:
    paths:
      - tfplan
  environment:
    name: dev
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

apply-dev:
  stage: apply
  image: hashicorp/terraform:${TF_VERSION}
  script:
    - terraform init
    - terraform apply -auto-approve tfplan
  environment:
    name: dev
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

plan-prod:
  stage: plan
  image: hashicorp/terraform:${TF_VERSION}
  script:
    - terraform init
    - terraform plan -out=tfplan -var-file=environments/prod.tfvars
  artifacts:
    paths:
      - tfplan
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

apply-prod:
  stage: apply
  image: hashicorp/terraform:${TF_VERSION}
  script:
    - terraform init
    - terraform apply -auto-approve tfplan
  environment:
    name: production
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  dependencies:
    - plan-prod
```

### 8. Pre-commit Hooks

**File: `.pre-commit-config.yaml`**
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: detect-private-key

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.77.0
    hooks:
      - id: terraform_fmt
        args:
          - --args=-recursive
      - id: terraform_docs
        args:
          - --args=--output-file=README.md
          - --args=--output-mode=inject
      - id: terraform_tflint
        args:
          - --args=--config=.tflint.hcl
      - id: terraform_validate
      - id: terraform_trivy
        args:
          - --args=--severity HIGH,CRITICAL

  - repo: https://github.com/bridgecrewio/checkov
    rev: 2.3.100
    hooks:
      - id: checkov
        args:
          - --framework=terraform
          - --quiet
```

**Setup:**
```bash
# Install pre-commit
pip install pre-commit

# Install hooks
pre-commit install

# Run manually
pre-commit run --all-files
```

### 9. Documentation

#### Automated Documentation with terraform-docs

```bash
# Install
brew install terraform-docs

# Generate documentation
terraform-docs markdown table . > README.md

# Generate specific sections
terraform-docs markdown table --output-file README.md --output-mode inject .

# Configuration: .terraform-docs.yml
formatter: "markdown table"

version: ""

header-from: main.tf

recursive:
  enabled: false
  path: modules

output:
  file: README.md
  mode: inject
  template: |-
    <!-- BEGIN_TF_DOCS -->
    {{ .Content }}
    <!-- END_TF_DOCS -->

content: |-
  {{ .Header }}
  
  ## Requirements
  
  {{ .Requirements }}
  
  ## Providers
  
  {{ .Providers }}
  
  ## Modules
  
  {{ .Modules }}
  
  ## Resources
  
  {{ .Resources }}
  
  ## Inputs
  
  {{ .Inputs }}
  
  ## Outputs
  
  {{ .Outputs }}
```

#### README Template

```markdown
# Infrastructure Module

Description of what this module does.

## Usage

```hcl
module "example" {
  source = "./modules/networking"
  
  vpc_cidr = "10.0.0.0/16"
  environment = "prod"
}
```

## Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.0.0 |
| aws | ~> 5.0 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| vpc_cidr | CIDR block for VPC | `string` | `"10.0.0.0/16"` | no |
| environment | Environment name | `string` | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | The ID of the VPC |
| subnet_ids | List of subnet IDs |
```

### 10. Performance Optimization

#### Use -parallelism Flag

```bash
# Default parallelism is 10
terraform apply -parallelism=50

# For large deployments
terraform apply -parallelism=250
```

#### Target Specific Resources

```bash
# Apply only specific resources
terraform apply -target=aws_instance.web
terraform apply -target=module.vpc

# Useful for fixing broken states
```

#### Use refresh=false

```bash
# Skip refresh for faster plans
terraform plan -refresh=false

# When you know state hasn't changed
```

#### Module Caching

```bash
# Terraform automatically caches modules
# Location: ~/.terraform/modules

# Clear cache if needed
rm -rf ~/.terraform/modules
```

---

## 🧪 Hands-On Lab: Complete CI/CD Pipeline

### Objective
Set up a complete CI/CD pipeline for Terraform with validation, planning, and deployment.

### Step 1: Create Repository Structure

```bash
mkdir -p terraform-cicd/{environments,modules/networking,scripts}
cd terraform-cicd

# Initialize git
git init
git remote add origin <your-repo-url>
```

### Step 2: Create Base Configuration

**File: `modules/networking/main.tf`**
```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-vpc"
  })
}

resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]
  
  map_public_ip_on_launch = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-public-${count.index + 1}"
    Type = "Public"
  })
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_ids" {
  value = aws_subnet.public[*].id
}
```

### Step 3: Create Environment Files

**File: `environments/dev.tfvars`**
```hcl
project_name   = "myapp-dev"
vpc_cidr       = "10.0.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b"]

common_tags = {
  Environment = "dev"
  ManagedBy   = "Terraform"
}
```

### Step 4: Setup Pre-commit Hooks

```bash
# Create .pre-commit-config.yaml (from above)

# Install and configure
pip install pre-commit
pre-commit install
pre-commit autoupdate

# Test
pre-commit run --all-files
```

### Step 5: Create GitHub Actions Workflow

(Use the workflow from section 7)

### Step 6: Test Locally

```bash
# Initialize
terraform init

# Validate
terraform validate

# Format check
terraform fmt -check -recursive

# Plan
terraform plan -var-file=environments/dev.tfvars

# Apply (for testing)
terraform apply -var-file=environments/dev.tfvars -auto-approve

# Destroy
terraform destroy -var-file=environments/dev.tfvars -auto-approve
```

### Step 7: Push and Trigger CI/CD

```bash
git add .
git commit -m "Initial Terraform configuration with CI/CD"
git push origin main

# Watch GitHub Actions tab for pipeline execution
```

---

## 📝 Summary

✅ **Code Organization**: Proper directory structure and naming  
✅ **Variable Management**: Types, validation, sensitive data  
✅ **State Management**: Remote backends, isolation strategies  
✅ **Security**: Least privilege, encryption, secrets management  
✅ **Testing**: validate, fmt, tflint, Terratest  
✅ **CI/CD**: GitHub Actions, GitLab CI pipelines  
✅ **Pre-commit**: Automated quality checks  
✅ **Documentation**: terraform-docs, README templates  
✅ **Performance**: Parallelism, targeting, caching  

Following these best practices will make your Terraform code production-ready, secure, and maintainable!

Next: **Module 08 - Monitoring & Observability** (Coming Soon)
