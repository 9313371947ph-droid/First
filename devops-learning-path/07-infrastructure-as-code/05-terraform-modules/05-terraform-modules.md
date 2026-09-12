# Module 07: Infrastructure as Code
## Section 05: Terraform Modules - Reusability at Scale

### 1. Introduction to Terraform Modules

#### What is a Terraform Module?
A **Terraform module** is a container for multiple resources that are used together. A module consists of a collection of `.tf` and/or `.tf.json` files kept together in a directory. Modules are the primary mechanism for code reuse in Terraform.

**Key Benefits:**
- **Reusability**: Write once, use multiple times
- **Maintainability**: Update in one place, propagate everywhere
- **Abstraction**: Hide complexity behind simple interfaces
- **Consistency**: Enforce organizational standards
- **Testing**: Test modules independently before deployment

#### Module Hierarchy
```
Root Module
├── VPC Module (source = "terraform-aws-modules/vpc/aws")
├── EKS Module (source = "terraform-aws-modules/eks/aws")
└── RDS Module (source = "terraform-aws-modules/rds/aws")
```

Every Terraform configuration has at least one module, known as its **root module**. This module calls other **child modules**, which can themselves call other modules.

### 2. Creating Your First Module

#### Module Structure
A basic module follows this structure:
```
modules/
└── web-server/
    ├── main.tf      # Resources
    ├── variables.tf # Input variables
    ├── outputs.tf   # Output values
    ├── versions.tf  # Provider version constraints
    └── README.md    # Documentation
```

#### Example: Simple Web Server Module

**File: `modules/web-server/main.tf`**
```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = {
    Name        = var.instance_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>${var.instance_name}</h1>" > /var/www/html/index.html
              EOF
}

resource "aws_security_group" "web_sg" {
  name        = "${var.instance_name}-sg"
  description = "Security group for web server"
  vpc_id      = var.vpc_id

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

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.instance_name}-sg"
  }
}
```

**File: `modules/web-server/variables.tf`**
```hcl
variable "ami_id" {
  description = "AMI ID for the EC2 instance"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "instance_name" {
  description = "Name tag for the instance"
  type        = string
}

variable "environment" {
  description = "Environment (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "vpc_id" {
  description = "VPC ID where the instance will be launched"
  type        = string
}

variable "subnet_id" {
  description = "Subnet ID for the instance"
  type        = string
}
```

**File: `modules/web-server/outputs.tf`**
```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "security_group_id" {
  description = "ID of the security group"
  value       = aws_security_group.web_sg.id
}
```

**File: `modules/web-server/versions.tf`**
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
```

### 3. Using Modules in Root Configuration

#### Calling a Local Module

**File: `main.tf` (Root Module)**
```hcl
provider "aws" {
  region = "us-east-1"
}

module "web_server_dev" {
  source = "./modules/web-server"
  
  ami_id        = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  instance_name = "web-server-dev"
  environment   = "development"
  vpc_id        = "vpc-12345678"
  subnet_id     = "subnet-12345678"
  
  tags = {
    Project = "DevOps Learning"
  }
}

module "web_server_prod" {
  source = "./modules/web-server"
  
  ami_id        = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.large"
  instance_name = "web-server-prod"
  environment   = "production"
  vpc_id        = "vpc-12345678"
  subnet_id     = "subnet-87654321"
  
  tags = {
    Project = "DevOps Learning"
  }
}

output "dev_instance_id" {
  value = module.web_server_dev.instance_id
}

output "prod_public_ip" {
  value = module.web_server_prod.public_ip
}
```

#### Accessing Module Outputs
```hcl
output "all_instances" {
  value = {
    dev_id  = module.web_server_dev.instance_id
    dev_ip  = module.web_server_dev.public_ip
    prod_id = module.web_server_prod.instance_id
    prod_ip = module.web_server_prod.public_ip
  }
}
```

### 4. Module Sources

Terraform supports multiple module sources:

#### Local Paths
```hcl
module "local_module" {
  source = "./modules/vpc"
}

module "relative_module" {
  source = "../../modules/shared-network"
}
```

#### Terraform Registry
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"  # Always specify version!
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}
```

#### GitHub
```hcl
module "ecs_cluster" {
  source = "github.com/terraform-aws-modules/terraform-aws-ecs-cluster?ref=v2.0.0"
  
  cluster_name = "my-cluster"
}
```

#### Git (SSH)
```hcl
module "private_module" {
  source = "git@github.com:myorg/terraform-modules.git//path/to/module?ref=v1.0.0"
}
```

#### Bitbucket
```hcl
module "bitbucket_module" {
  source = "bitbucket.org/myorg/myrepo//modules/network?ref=v1.0.0"
}
```

#### S3 Bucket
```hcl
module "s3_module" {
  source = "s3::https://my-bucket.s3.amazonaws.com/modules/vpc.zip"
}
```

#### HTTP URL
```hcl
module "http_module" {
  source = "https://example.com/modules/vpc.zip"
}
```

### 5. Best Practices for Module Development

#### Version Pinning
Always pin module versions to avoid unexpected changes:
```hcl
# ✅ Good
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

# ❌ Bad - No version constraint
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
}
```

#### Semantic Versioning
Follow SemVer for your modules:
- **MAJOR.MINOR.PATCH** (e.g., 2.1.3)
- MAJOR: Breaking changes
- MINOR: New features (backward compatible)
- PATCH: Bug fixes (backward compatible)

#### Input Validation
```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}
```

#### Consistent Naming Conventions
```hcl
# Use snake_case for resources and variables
resource "aws_instance" "web_server" {}
variable "instance_type" {}

# Use descriptive names
output "database_endpoint" {}  # ✅ Clear
output "endpoint" {}           # ❌ Ambiguous
```

#### Documentation
Create comprehensive README.md:
```markdown
# AWS VPC Module

Terraform module which creates VPC resources on AWS.

## Usage

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| name | Name of the VPC | `string` | n/a | yes |
| cidr | CIDR block for VPC | `string` | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | The ID of the VPC |
| vpc_arn | The ARN of the VPC |
```

### 6. Advanced Module Patterns

#### Conditional Resources
```hcl
variable "enable_nat_gateway" {
  description = "Whether to create NAT Gateway"
  type        = bool
  default     = false
}

resource "aws_nat_gateway" "this" {
  count = var.enable_nat_gateway ? 1 : 0
  
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}

resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? 1 : 0
  
  domain = "vpc"
}
```

#### Dynamic Blocks
```hcl
variable "ingress_rules" {
  description = "List of ingress rules"
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = []
}

resource "aws_security_group" "example" {
  name = "example-sg"
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

#### Module Composition
```hcl
# Compose multiple modules together
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "app-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b"]
  
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"
  
  cluster_name    = "my-cluster"
  cluster_version = "1.27"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  eks_managed_node_groups = {
    default = {
      desired_size = 2
      min_size     = 1
      max_size     = 5
    }
  }
}
```

### 7. Publishing Modules to Terraform Registry

#### Requirements for Public Registry
1. Repository must be on GitHub
2. Follow naming convention: `terraform-PROVIDER-NAME`
3. Include proper documentation
4. Tag releases with semantic versions

#### Steps to Publish
```bash
# 1. Create repository with proper name
git clone git@github.com:username/terraform-aws-web-server.git
cd terraform-aws-web-server

# 2. Add module files
# (main.tf, variables.tf, outputs.tf, README.md, LICENSE)

# 3. Create initial commit
git add .
git commit -m "Initial release"
git push origin main

# 4. Tag the release
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# 5. Module is automatically published to registry
# Visit: https://registry.terraform.io/modules/username/web-server/aws
```

#### Private Registry (Enterprise)
```hcl
module "private_module" {
  source = "app.terraform.io/my-org/web-server/aws"
  version = "1.0.0"
}
```

### 8. Testing Modules

#### Using terraform-docs
Generate documentation automatically:
```bash
# Install
brew install terraform-docs

# Generate README
terraform-docs markdown table . >> README.md

# Generate specific format
terraform-docs json . > module-metadata.json
```

#### Using tflint for Linting
```bash
# Install
brew install tflint

# Initialize plugins
tflint --init

# Run linting
tflint
```

#### Using pre-commit hooks
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.77.0
    hooks:
      - id: terraform_fmt
      - id: terraform_docs
      - id: terraform_tflint
      - id: terraform_trivy
```

### 9. Common Module Patterns

#### Multi-Environment Module
```hcl
# environments/dev/main.tf
module "web_app" {
  source = "../../modules/web-app"
  
  environment      = "dev"
  instance_count   = 1
  instance_type    = "t3.micro"
  enable_monitoring = false
}

# environments/prod/main.tf
module "web_app" {
  source = "../../modules/web-app"
  
  environment      = "prod"
  instance_count   = 5
  instance_type    = "t3.xlarge"
  enable_monitoring = true
}
```

#### Wrapper Module Pattern
```hcl
# Create opinionated wrapper around complex module
module "company_vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  # Enforce company standards
  name = "${var.project_name}-vpc"
  cidr = var.company_cidr_block
  
  # Pre-configured subnets
  azs             = data.aws_availability_zones.available.names
  private_subnets = [for k, v in data.aws_availability_zones.available.names : cidrsubnet(var.company_cidr_block, 8, k)]
  public_subnets  = [for k, v in data.aws_availability_zones.available.names : cidrsubnet(var.company_cidr_block, 8, k + 100)]
  
  # Enable required features
  enable_nat_gateway = true
  single_nat_gateway = var.environment != "prod"
  
  # Security tags
  tags = merge(var.tags, {
    Compliance = "SOC2"
    Owner      = "Platform Team"
  })
}
```

### 10. Troubleshooting Common Issues

#### Issue: Module Source Not Found
```
Error: Failed to download module
│ Could not download module "vpc" ...
```
**Solution**: Check source path, network connectivity, and authentication.

#### Issue: Version Constraint Conflicts
```
Error: Failed to select appropriate version
│ No versions of module satisfy the version constraints
```
**Solution**: Relax version constraints or update module versions.

#### Issue: Variable Type Mismatch
```
Error: Invalid value for variable
│ string required
```
**Solution**: Ensure variable types match expected types in module.

---

## 🧪 Hands-On Lab: Create and Use a Custom Module

### Lab Objective
Create a reusable RDS module and deploy it in multiple environments.

### Step 1: Create Module Directory Structure
```bash
mkdir -p modules/rds-mysql/{examples,tests}
touch modules/rds-mysql/{main.tf,variables.tf,outputs.tf,versions.tf,README.md}
```

### Step 2: Implement RDS Module

**File: `modules/rds-mysql/main.tf`**
```hcl
resource "aws_db_subnet_group" "main" {
  name       = "${var.db_name}-subnet-group"
  subnet_ids = var.subnet_ids
  
  tags = merge(var.tags, {
    Name = "${var.db_name}-subnet-group"
  })
}

resource "aws_security_group" "rds" {
  name        = "${var.db_name}-sg"
  description = "Security group for RDS ${var.db_name}"
  vpc_id      = var.vpc_id
  
  ingress {
    from_port   = var.db_port
    to_port     = var.db_port
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(var.tags, {
    Name = "${var.db_name}-sg"
  })
}

resource "aws_db_instance" "main" {
  identifier        = var.db_name
  engine            = var.engine
  engine_version    = var.engine_version
  instance_class    = var.instance_class
  allocated_storage = var.allocated_storage
  storage_type      = var.storage_type
  
  db_name  = var.database_name
  username = var.db_username
  password = var.db_password
  
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  
  multi_az               = var.multi_az
  publicly_accessible    = var.publicly_accessible
  backup_retention_period = var.backup_retention_period
  skip_final_snapshot    = var.skip_final_snapshot
  
  tags = merge(var.tags, {
    Name        = var.db_name
    Environment = var.environment
  })
}
```

**File: `modules/rds-mysql/variables.tf`**
```hcl
variable "db_name" {
  description = "Name of the RDS instance"
  type        = string
}

variable "engine" {
  description = "Database engine"
  type        = string
  default     = "mysql"
}

variable "engine_version" {
  description = "Database engine version"
  type        = string
  default     = "8.0"
}

variable "instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "allocated_storage" {
  description = "Allocated storage in GB"
  type        = number
  default     = 20
}

variable "storage_type" {
  description = "Storage type"
  type        = string
  default     = "gp2"
}

variable "database_name" {
  description = "Name of the database"
  type        = string
  default     = "appdb"
}

variable "db_username" {
  description = "Master username"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "Master password"
  type        = string
  sensitive   = true
}

variable "subnet_ids" {
  description = "List of subnet IDs"
  type        = list(string)
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "allowed_cidr_blocks" {
  description = "CIDR blocks allowed to access the database"
  type        = list(string)
  default     = []
}

variable "multi_az" {
  description = "Enable Multi-AZ deployment"
  type        = bool
  default     = false
}

variable "publicly_accessible" {
  description = "Make database publicly accessible"
  type        = bool
  default     = false
}

variable "backup_retention_period" {
  description = "Backup retention period in days"
  type        = number
  default     = 7
}

variable "skip_final_snapshot" {
  description = "Skip final snapshot when deleting"
  type        = bool
  default     = true
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "tags" {
  description = "Tags to apply"
  type        = map(string)
  default     = {}
}
```

**File: `modules/rds-mysql/outputs.tf`**
```hcl
output "db_endpoint" {
  description = "The connection endpoint"
  value       = aws_db_instance.main.endpoint
}

output "db_address" {
  description = "The hostname of the RDS instance"
  value       = aws_db_instance.main.address
}

output "db_port" {
  description = "The database port"
  value       = aws_db_instance.main.port
}

output "db_name" {
  description = "The database name"
  value       = aws_db_instance.main.db_name
}

output "db_username" {
  description = "The master username"
  value       = aws_db_instance.main.username
  sensitive   = true
}

output "security_group_id" {
  description = "The security group ID"
  value       = aws_security_group.rds.id
}
```

### Step 3: Use Module in Root Configuration

**File: `environments/dev/main.tf`**
```hcl
provider "aws" {
  region = "us-east-1"
}

module "rds_dev" {
  source = "../../modules/rds-mysql"
  
  db_name         = "app-db-dev"
  instance_class  = "db.t3.micro"
  allocated_storage = 20
  
  database_name = "devdb"
  db_username   = "devuser"
  db_password   = "devpassword123!"  # Use secrets manager in production!
  
  subnet_ids          = ["subnet-dev-1", "subnet-dev-2"]
  vpc_id              = "vpc-dev-123"
  allowed_cidr_blocks = ["10.0.0.0/16"]
  
  environment = "dev"
  skip_final_snapshot = true
  
  tags = {
    Project = "DevOps Learning"
  }
}

output "dev_db_endpoint" {
  value = module.rds_dev.db_endpoint
}
```

### Step 4: Deploy and Verify
```bash
cd environments/dev
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# Verify output
echo "Database Endpoint: $(terraform output -raw dev_db_endpoint)"
```

### Step 5: Deploy to Production
```bash
cd ../prod
terraform init
terraform plan -out=tfplan -var="db_password=$(aws secretsmanager get-secret-value --secret-id prod-db-password --query SecretString --output text)"
terraform apply tfplan
```

---

## 📝 Summary

✅ **Module Fundamentals**: Understanding what modules are and why they matter  
✅ **Creating Modules**: Building reusable components with inputs and outputs  
✅ **Module Sources**: Local paths, registry, GitHub, and more  
✅ **Best Practices**: Versioning, documentation, testing  
✅ **Advanced Patterns**: Conditionals, dynamic blocks, composition  
✅ **Publishing**: Sharing modules via Terraform Registry  
✅ **Lab**: Created and deployed a custom RDS module  

Modules are the foundation of scalable Infrastructure as Code. Master them to build maintainable, reusable, and enterprise-grade Terraform configurations!

Next: **Section 06 - Terraform Workspaces & Environments**
