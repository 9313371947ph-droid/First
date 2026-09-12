# Module 07: Infrastructure as Code
## Section 06: Terraform Workspaces & Environment Management

### 1. Introduction to Terraform Workspaces

#### What are Workspaces?
**Terraform workspaces** allow you to create multiple instances of a configuration within the same working directory. Each workspace has its own state file, enabling you to manage different environments (dev, staging, prod) or separate deployments with the same codebase.

**Key Use Cases:**
- **Environment Separation**: dev, staging, production
- **Team Isolation**: Different teams working on same infrastructure
- **Feature Testing**: Test changes without affecting main environment
- **Regional Deployments**: Same config deployed to multiple regions
- **Cost Optimization**: Reduce code duplication across environments

#### Workspace vs State File
```
Default Behavior:
└── terraform.tfstate

With Workspaces:
└── env:/
    ├── dev/
    │   └── terraform.tfstate
    ├── staging/
    │   └── terraform.tfstate
    └── prod/
        └── terraform.tfstate
```

### 2. Basic Workspace Operations

#### Creating and Selecting Workspaces

```bash
# List all workspaces
terraform workspace list

# Create a new workspace
terraform workspace new dev

# Create and switch to new workspace
terraform workspace new staging

# Switch to existing workspace
terraform workspace select prod

# Show current workspace
terraform workspace show

# Delete a workspace (must not be current)
terraform workspace delete old-dev

# Output the current workspace name
echo $(terraform workspace show)
```

#### Default Workspace
Every Terraform configuration starts with a `default` workspace:
```bash
# You're always in a workspace
terraform workspace show  # Outputs: default

# The default workspace cannot be deleted
terraform workspace delete default  # ❌ Error
```

### 3. Using Workspaces in Configuration

#### Conditional Resources Based on Workspace

**File: `main.tf`**
```hcl
provider "aws" {
  region = "us-east-1"
}

# Get current workspace name
locals {
  environment = terraform.workspace
  is_prod     = terraform.workspace == "prod"
  is_dev      = terraform.workspace == "dev"
}

# Instance type varies by environment
variable "instance_types" {
  type = map(string)
  default = {
    dev      = "t3.micro"
    staging  = "t3.small"
    prod     = "t3.xlarge"
    default  = "t3.micro"
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = lookup(var.instance_types, terraform.workspace, var.instance_types["default"])
  
  tags = {
    Name        = "web-server-${terraform.workspace}"
    Environment = terraform.workspace
    ManagedBy   = "Terraform"
  }
}

# Multi-AZ only for production
resource "aws_db_instance" "database" {
  count = local.is_prod ? 1 : 0
  
  identifier        = "app-db-${terraform.workspace}"
  engine            = "mysql"
  engine_version    = "8.0"
  instance_class    = local.is_prod ? "db.r5.large" : "db.t3.micro"
  allocated_storage = local.is_prod ? 100 : 20
  
  multi_az = local.is_prod
  
  tags = {
    Environment = terraform.workspace
  }
}

# Different security groups per environment
resource "aws_security_group" "web_sg" {
  name = "web-sg-${terraform.workspace}"
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    # Restrict prod, open for dev
    cidr_blocks = local.is_prod ? ["10.0.0.0/8"] : ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Environment = terraform.workspace
  }
}

output "environment" {
  value = terraform.workspace
}

output "instance_id" {
  value = aws_instance.web.id
}

output "database_endpoint" {
  value = local.is_prod ? aws_db_instance.database[0].endpoint : "No database in non-prod environments"
}
```

#### Workspace-Specific Variables

**File: `variables.tf`**
```hcl
variable "environment_config" {
  description = "Configuration per environment"
  type = map(object({
    instance_count  = number
    instance_type   = string
    enable_monitoring = bool
    backup_retention = number
  }))
  
  default = {
    dev = {
      instance_count   = 1
      instance_type    = "t3.micro"
      enable_monitoring = false
      backup_retention = 1
    }
    staging = {
      instance_count   = 2
      instance_type    = "t3.small"
      enable_monitoring = true
      backup_retention = 7
    }
    prod = {
      instance_count   = 5
      instance_type    = "t3.xlarge"
      enable_monitoring = true
      backup_retention = 30
    }
  }
}

locals {
  env_config = lookup(var.environment_config, terraform.workspace, var.environment_config["dev"])
}

resource "aws_autoscaling_group" "app" {
  desired_capacity = local.env_config.instance_count
  max_size         = local.env_config.instance_count * 2
  min_size         = local.env_config.instance_count
  
  launch_configuration = aws_launch_configuration.app.name
  
  tag {
    key                 = "Environment"
    value               = terraform.workspace
    propagate_at_launch = true
  }
}
```

### 4. Workspace State Management

#### State File Locations

**Local Backend:**
```
.tfstate.d/
├── dev/
│   └── terraform.tfstate
├── staging/
│   └── terraform.tfstate
└── prod/
    └── terraform.tfstate
```

**Remote Backend (S3):**
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "my-app/terraform.tfstate"
    region = "us-east-1"
    
    # Workspaces stored as prefixes
    # dev/terraform.tfstate
    # staging/terraform.tfstate
    # prod/terraform.tfstate
  }
}
```

#### Viewing Workspace State
```bash
# List resources in current workspace
terraform state list

# Show specific resource
terraform state show aws_instance.web

# Compare workspaces
terraform workspace select dev
terraform output > dev-output.txt

terraform workspace select prod
terraform output > prod-output.txt

diff dev-output.txt prod-output.txt
```

### 5. Advanced Workspace Patterns

#### Pattern 1: Environment-Specific Modules
```hcl
locals {
  common_tags = {
    Project     = "MyApp"
    ManagedBy   = "Terraform"
    Environment = terraform.workspace
  }
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "app-vpc-${terraform.workspace}"
  cidr = "10.0.0.0/16"
  
  azs = ["us-east-1a", "us-east-1b"]
  
  # Different subnet counts per environment
  private_subnets = terraform.workspace == "prod" 
    ? ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
    : ["10.0.1.0/24"]
  
  public_subnets = terraform.workspace == "prod"
    ? ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
    : ["10.0.101.0/24"]
  
  enable_nat_gateway = terraform.workspace == "prod"
  single_nat_gateway = terraform.workspace != "prod"
  
  tags = local.common_tags
}
```

#### Pattern 2: Workspace-Aware Naming
```hcl
locals {
  name_prefix = "${var.project_name}-${terraform.workspace}"
  
  naming_convention = {
    dev      = "${local.name_prefix}-dev"
    staging  = "${local.name_prefix}-stg"
    prod     = "${local.name_prefix}-prd"
    default  = local.name_prefix
  }
  
  resource_name = lookup(local.naming_convention, terraform.workspace, local.name_prefix)
}

resource "aws_s3_bucket" "data" {
  bucket = "${local.resource_name}-data-bucket"
  
  tags = {
    Name        = local.resource_name
    Environment = terraform.workspace
  }
}
```

#### Pattern 3: Conditional Module Deployment
```hcl
# Only deploy monitoring in prod
module "datadog" {
  count  = terraform.workspace == "prod" ? 1 : 0
  source = "./modules/monitoring"
  
  api_key = var.datadog_api_key
  env     = terraform.workspace
}

# Deploy prometheus in all environments
module "prometheus" {
  source = "./modules/prometheus"
  
  env = terraform.workspace
  retention_days = terraform.workspace == "prod" ? 90 : 7
}
```

### 6. Workspace Best Practices

#### DO ✅
- Use workspaces for similar environments with same codebase
- Combine with remote backends for team collaboration
- Use `terraform.workspace` in tags and naming
- Document which workspaces exist and their purpose
- Implement CI/CD pipelines per workspace

#### DON'T ❌
- Don't use workspaces for completely different projects
- Don't rely solely on workspaces for access control
- Don't mix vastly different configurations in one codebase
- Don't forget to switch workspaces before applying
- Don't use workspaces as a substitute for proper state isolation

#### Workspace Naming Convention
```bash
# Good naming patterns
terraform workspace new dev-us-east-1
terraform workspace new prod-eu-west-1
terraform workspace new staging-feature-auth

# Bad naming patterns
terraform workspace new test1
terraform workspace new myworkspace
terraform workspace new production-backup-old
```

### 7. CI/CD Integration with Workspaces

#### GitHub Actions Example

**File: `.github/workflows/terraform.yml`**
```yaml
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  TF_VERSION: 1.5.0

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        workspace: [dev, staging, prod]
    
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
      
      - name: Select Workspace
        run: terraform workspace select ${{ matrix.workspace }} || terraform workspace new ${{ matrix.workspace }}
      
      - name: Terraform Format Check
        run: terraform fmt -check
      
      - name: Terraform Plan
        run: terraform plan -out=tfplan -input=false
        env:
          TF_VAR_environment: ${{ matrix.workspace }}
      
      - name: Terraform Apply (Dev Only)
        if: github.ref == 'refs/heads/main' && matrix.workspace == 'dev'
        run: terraform apply -auto-approve tfplan
      
      - name: Manual Approval for Prod
        if: matrix.workspace == 'prod'
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: admin-user
          minimum-approvals: 1
          issue-title: "Deploy to Production?"
          issue-body: "Please review and approve the deployment to production"
          exclude-workflow-initiator-as-approver: true
      
      - name: Terraform Apply (Prod After Approval)
        if: matrix.workspace == 'prod' && github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan
```

#### Atlantis Integration
```yaml
# atlantis.yaml
version: 3
projects:
  - name: dev
    dir: .
    workspace: dev
    autoplan:
      enabled: true
      when_modified: ["*.tf"]
    apply_requirements: [approved]
  
  - name: staging
    dir: .
    workspace: staging
    autoplan:
      enabled: true
      when_modified: ["*.tf"]
    apply_requirements: [approved]
  
  - name: prod
    dir: .
    workspace: prod
    autoplan:
      enabled: false
    apply_requirements: [approved, mergeable]
```

### 8. Troubleshooting Workspaces

#### Common Issues

**Issue 1: Wrong Workspace Selected**
```bash
# Check current workspace
terraform workspace show

# Switch to correct workspace
terraform workspace select prod

# Verify state
terraform state list
```

**Issue 2: State Not Found**
```
Error: Failed to load state: no state files were found
```
**Solution**: Ensure workspace exists and has been initialized:
```bash
terraform workspace new dev
terraform apply
```

**Issue 3: Cannot Delete Current Workspace**
```
Error: Cannot delete the currently selected workspace
```
**Solution**: Switch to another workspace first:
```bash
terraform workspace select default
terraform workspace delete dev
```

**Issue 4: Workspace-Specific Variables Not Applied**
```hcl
# Ensure you're using terraform.workspace correctly
locals {
  config = lookup(var.env_configs, terraform.workspace, {})
}
```

### 9. Migration Strategies

#### From Separate Directories to Workspaces

**Before (Directory-based):**
```
environments/
├── dev/
│   ├── main.tf
│   └── terraform.tfstate
├── staging/
│   ├── main.tf
│   └── terraform.tfstate
└── prod/
    ├── main.tf
    └── terraform.tfstate
```

**After (Workspace-based):**
```
infrastructure/
├── main.tf
├── variables.tf
└── .terraform/
    └── environments/
        ├── dev/
        │   └── terraform.tfstate
        ├── staging/
        │   └── terraform.tfstate
        └── prod/
            └── terraform.tfstate
```

**Migration Steps:**
```bash
# 1. Backup existing states
cp -r environments environments-backup

# 2. Consolidate configurations
# Merge dev/staging/prod main.tf into single main.tf with workspace logic

# 3. Initialize workspaces
terraform init
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# 4. Import existing resources (if needed)
terraform workspace select dev
terraform import aws_instance.web i-1234567890abcdef0

# 5. Verify and clean up
terraform plan
# Once verified, remove old directories
rm -rf environments
```

---

## 🧪 Hands-On Lab: Multi-Environment Deployment with Workspaces

### Lab Objective
Deploy a web application to three environments (dev, staging, prod) using workspaces.

### Step 1: Create Base Configuration

**File: `main.tf`**
```hcl
terraform {
  required_version = ">= 1.0.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

locals {
  environment = terraform.workspace
  
  env_configs = {
    dev = {
      instance_type    = "t3.micro"
      instance_count   = 1
      enable_monitoring = false
      db_size          = "db.t3.micro"
    }
    staging = {
      instance_type    = "t3.small"
      instance_count   = 2
      enable_monitoring = true
      db_size          = "db.t3.small"
    }
    prod = {
      instance_type    = "t3.medium"
      instance_count   = 3
      enable_monitoring = true
      db_size          = "db.r5.large"
    }
  }
  
  config = lookup(local.env_configs, local.environment, local.env_configs["dev"])
}

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name        = "app-vpc-${local.environment}"
    Environment = local.environment
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name        = "app-igw-${local.environment}"
    Environment = local.environment
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
  
  tags = {
    Name        = "app-subnet-public-${local.environment}"
    Environment = local.environment
  }
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name        = "app-rt-public-${local.environment}"
    Environment = local.environment
  }
}

# Security Group
resource "aws_security_group" "web" {
  name        = "web-sg-${local.environment}"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # Only from VPC
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name        = "web-sg-${local.environment}"
    Environment = local.environment
  }
}

# Launch Template
resource "aws_launch_template" "web" {
  name_prefix   = "web-lt-${local.environment}-"
  image_id      = "ami-0c55b159cbfafe1f0"
  instance_type = local.config.instance_type
  
  network_interfaces {
    associate_public_ip_address = true
    security_groups             = [aws_security_group.web.id]
  }
  
  user_data = base64encode(<<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from ${local.environment}!</h1>" > /var/www/html/index.html
              echo "<p>Environment: ${local.environment}</p>" >> /var/www/html/index.html
              EOF
  )
  
  tag_specifications {
    resource_type = "instance"
    
    tags = {
      Name        = "web-server-${local.environment}"
      Environment = local.environment
    }
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "web" {
  name                = "web-asg-${local.environment}"
  vpc_zone_identifier = aws_subnet.public.id
  target_group_arns   = [aws_lb_target_group.web.arn]
  health_check_type   = "ELB"
  
  min_size         = local.config.instance_count
  max_size         = local.config.instance_count * 2
  desired_capacity = local.config.instance_count
  
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
  
  tag {
    key                 = "Name"
    value               = "web-server-${local.environment}"
    propagate_at_launch = true
  }
  
  tag {
    key                 = "Environment"
    value               = local.environment
    propagate_at_launch = true
  }
}

# Application Load Balancer
resource "aws_lb" "web" {
  name               = "web-alb-${local.environment}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web.id]
  subnets            = [aws_subnet.public.id]
  
  enable_deletion_protection = local.environment == "prod"
  
  tags = {
    Name        = "web-alb-${local.environment}"
    Environment = local.environment
  }
}

resource "aws_lb_target_group" "web" {
  name     = "web-tg-${local.environment}"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  
  health_check {
    path                = "/"
    healthy_threshold   = 2
    unhealthy_threshold = 10
  }
  
  tags = {
    Name        = "web-tg-${local.environment}"
    Environment = local.environment
  }
}

resource "aws_lb_listener" "web" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  protocol          = "HTTP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# RDS Database (only for staging and prod)
resource "aws_db_instance" "app" {
  count = local.environment != "dev" ? 1 : 0
  
  identifier        = "app-db-${local.environment}"
  engine            = "mysql"
  engine_version    = "8.0"
  instance_class    = local.config.db_size
  
  allocated_storage = local.environment == "prod" ? 100 : 20
  storage_type      = "gp2"
  
  db_name  = "appdb"
  username = "admin"
  password = "SecurePassword123!"  # Use secrets manager in real scenario!
  
  vpc_security_group_ids = [aws_security_group.web.id]
  db_subnet_group_name   = aws_db_subnet_group.app[0].name
  
  multi_az              = local.environment == "prod"
  publicly_accessible   = false
  skip_final_snapshot   = local.environment != "prod"
  backup_retention_period = local.environment == "prod" ? 30 : 7
  
  tags = {
    Name        = "app-db-${local.environment}"
    Environment = local.environment
  }
}

resource "aws_db_subnet_group" "app" {
  count = local.environment != "dev" ? 1 : 0
  
  name       = "app-db-subnet-${local.environment}"
  subnet_ids = [aws_subnet.public.id]  # In production, use private subnets!
  
  tags = {
    Name        = "app-db-subnet-${local.environment}"
    Environment = local.environment
  }
}

# Outputs
output "environment" {
  description = "Current environment"
  value       = local.environment
}

output "alb_dns_name" {
  description = "ALB DNS name"
  value       = aws_lb.web.dns_name
}

output "asg_name" {
  description = "Auto Scaling Group name"
  value       = aws_autoscaling_group.web.name
}

output "database_endpoint" {
  description = "RDS endpoint (if applicable)"
  value       = local.environment != "dev" ? aws_db_instance.app[0].endpoint : "No database in dev"
}

output "instance_count" {
  description = "Number of instances"
  value       = local.config.instance_count
}
```

### Step 2: Deploy to All Environments

```bash
# Initialize Terraform
terraform init

# Deploy to DEV
terraform workspace new dev || terraform workspace select dev
terraform plan -out=tfplan-dev
terraform apply tfplan-dev

echo "✅ DEV Environment Deployed"
echo "ALB URL: $(terraform output -raw alb_dns_name)"

# Deploy to STAGING
terraform workspace new staging || terraform workspace select staging
terraform plan -out=tfplan-staging
terraform apply tfplan-staging

echo "✅ STAGING Environment Deployed"
echo "ALB URL: $(terraform output -raw alb_dns_name)"

# Deploy to PROD
terraform workspace new prod || terraform workspace select prod
terraform plan -out=tfplan-prod
terraform apply tfplan-prod

echo "✅ PROD Environment Deployed"
echo "ALB URL: $(terraform output -raw alb_dns_name)"
```

### Step 3: Verify Deployments

```bash
# List all workspaces
terraform workspace list

# Check resources in each workspace
for env in dev staging prod; do
  echo "=== $env Environment ==="
  terraform workspace select $env
  terraform output
  echo ""
done

# Access applications
DEV_URL=$(terraform workspace select dev && terraform output -raw alb_dns_name)
STAGING_URL=$(terraform workspace select staging && terraform output -raw alb_dns_name)
PROD_URL=$(terraform workspace select prod && terraform output -raw alb_dns_name)

echo "Dev: http://$DEV_URL"
echo "Staging: http://$STAGING_URL"
echo "Prod: http://$PROD_URL"
```

### Step 4: Cleanup

```bash
# Destroy in reverse order (prod first)
terraform workspace select prod
terraform destroy -auto-approve

terraform workspace select staging
terraform destroy -auto-approve

terraform workspace select dev
terraform destroy -auto-approve

# Delete workspaces
terraform workspace select default
terraform workspace delete dev
terraform workspace delete staging
terraform workspace delete prod
```

---

## 📝 Summary

✅ **Workspace Fundamentals**: Understanding workspaces and state isolation  
✅ **Basic Operations**: Create, select, list, and delete workspaces  
✅ **Conditional Logic**: Using `terraform.workspace` in configurations  
✅ **State Management**: Local and remote backend workspace handling  
✅ **Advanced Patterns**: Environment-specific modules, naming, conditional deployment  
✅ **Best Practices**: When to use workspaces, naming conventions, CI/CD integration  
✅ **Troubleshooting**: Common issues and solutions  
✅ **Lab**: Deployed complete web application to dev, staging, and prod  

Workspaces provide a powerful mechanism for managing multiple environments with a single codebase. Master them to streamline your infrastructure deployments!

Next: **Section 07 - Terraform Best Practices & CI/CD Integration**
