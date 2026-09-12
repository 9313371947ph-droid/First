# Section 02: Terraform Core Concepts

## 🎯 Learning Objectives
By the end of this section, you will be able to:
- Understand Terraform's declarative approach
- Master providers and resources
- Work with Terraform state files
- Use data sources effectively
- Understand the Terraform workflow
- Write your first complete Terraform configuration

---

## 1. Terraform's Declarative Philosophy

### Imperative vs. Declarative (Revisited)

**Imperative** (like shell scripts):
```bash
# You tell it HOW to do things
aws ec2 run-instances --image-id ami-123
aws ec2 create-tags --resources i-abc123 --tags Key=Name,Value=Web
aws ec2 authorize-security-group-ingress --port 80
```

**Declarative** (Terraform):
```hcl
# You declare WHAT you want
resource "aws_instance" "web" {
  ami           = "ami-123"
  instance_type = "t2.micro"
  
  tags = {
    Name = "Web"
  }
}

resource "aws_security_group_rule" "http" {
  security_group_id = aws_security_group.web.id
  type              = "ingress"
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

**Key Difference:**
- **Imperative**: Run script twice = create two instances ❌
- **Declarative**: Apply twice = same single instance ✅

### Desired State Management

Terraform maintains a **desired state** in your code and works to make reality match that state.

```
┌─────────────────────────────────────────────────────────┐
│            TERRAFORM STATE MANAGEMENT                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Your Code (Desired State)                               │
│  ┌──────────────┐                                       │
│  │ 1 VPC        │                                       │
│  │ 2 Subnets    │                                       │
│  │ 3 EC2s       │                                       │
│  └──────────────┘                                       │
│         ↓                                                │
│  Terraform Plan (Compares)                               │
│         ↓                                                │
│  Current State (Reality)                                 │
│  ┌──────────────┐                                       │
│  │ 0 VPCs       │ ← Need to create 1                   │
│  │ 0 Subnets    │ ← Need to create 2                   │
│  │ 0 EC2s       │ ← Need to create 3                   │
│  └──────────────┘                                       │
│                                                          │
│  terraform apply → Creates missing resources             │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Providers

### What is a Provider?

A **provider** is a plugin that enables Terraform to interact with a specific API (cloud provider, SaaS, on-premises, etc.).

**Popular Providers:**
| Provider | Description | Documentation |
|----------|-------------|---------------|
| `aws` | Amazon Web Services | registry.terraform.io/providers/hashicorp/aws |
| `azurerm` | Microsoft Azure | registry.terraform.io/providers/hashicorp/azurerm |
| `google` | Google Cloud Platform | registry.terraform.io/providers/hashicorp/google |
| `kubernetes` | Kubernetes clusters | registry.terraform.io/providers/hashicorp/kubernetes |
| `docker` | Docker containers | registry.terraform.io/providers/kreuzwerker/docker |
| `github` | GitHub repositories | registry.terraform.io/providers/integrations/github |
| `vault` | HashiCorp Vault | registry.terraform.io/providers/hashicorp/vault |

### Configuring Providers

#### Basic Provider Configuration

```hcl
# AWS Provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Use latest 5.x version
    }
  }
  
  required_version = ">= 1.0.0"  # Terraform version requirement
}

provider "aws" {
  region = "us-east-1"
  
  # Authentication options (choose one):
  # 1. Environment variables: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
  # 2. Shared credentials file (~/.aws/credentials)
  # 3. IAM role (when running on EC2/ECS)
  # 4. Explicitly (not recommended for production):
  # access_key = "YOUR_ACCESS_KEY"
  # secret_key = "YOUR_SECRET_KEY"
}
```

#### Multiple Regions/Accounts

```hcl
# Primary AWS account
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

# Secondary AWS account
provider "aws" {
  alias  = "secondary"
  region = "us-west-2"
  
  # Different credentials for different account
  profile = "production-account"
}

# Use specific provider
resource "aws_instance" "primary_region" {
  provider = aws.primary
  ami      = "ami-12345678"
  # ...
}

resource "aws_instance" "secondary_region" {
  provider = aws.secondary
  ami      = "ami-87654321"
  # ...
}
```

#### Provider Version Constraints

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0, < 6.0"  # Range constraint
    }
    
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"  # Equivalent to >= 3.0, < 4.0
    }
    
    random = {
      source  = "hashicorp/random"
      version = "= 3.5.0"  # Exact version
    }
  }
}
```

**Version Constraint Operators:**
- `=` : Exact version
- `!=` : Not equal
- `>` , `>=` , `<` , `<=` : Comparison
- `~>` : Pessimistic constraint (allows patch updates)
  - `~> 2.0` = `>= 2.0, < 3.0`
  - `~> 2.1.0` = `>= 2.1.0, < 2.2.0`

---

## 3. Resources

### What is a Resource?

A **resource** is a component of your infrastructure (VM, network, database, DNS record, etc.).

### Resource Syntax

```hcl
resource "<TYPE>" "<LOCAL_NAME>" {
  <ATTRIBUTE> = <VALUE>
  <ATTRIBUTE> = <VALUE>
  ...
  
  # Meta-arguments (available for all resources)
  depends_on = [...]
  count      = ...
  for_each   = ...
  lifecycle  = { ... }
  provider   = ...
}
```

### Example: EC2 Instance

```hcl
resource "aws_instance" "web_server" {
  # Required attributes
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  # Optional attributes
  key_name      = "my-key-pair"
  
  # Network configuration
  subnet_id     = aws_subnet.main.id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  # User data (script runs on boot)
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
              EOF
  
  # Tags
  tags = {
    Name        = "web-server-1"
    Environment = "production"
    Project     = "my-app"
    ManagedBy   = "terraform"
  }
  
  # Metadata (read-only)
  # These are computed by AWS and available after creation
  # public_ip      = automatically assigned
  # private_ip     = automatically assigned
  # availability_zone = automatically assigned
}
```

### Referencing Resources

```hcl
# Create security group first
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTP traffic"
  vpc_id      = aws_vpc.main.id  # Reference another resource
  
  ingress {
    from_port   = 80
    to_port     = 80
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
    Name = "web-security-group"
  }
}

# Reference the security group in instance
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  # Reference security group by its attribute
  vpc_security_group_ids = [aws_security_group.web.id]
  
  # Reference other attributes
  subnet_id = aws_subnet.public.id
}

# Output the instance's public IP
output "web_server_ip" {
  value = aws_instance.web.public_ip
}
```

### Resource Dependencies

Terraform automatically detects dependencies when you reference one resource in another.

**Explicit Dependencies** (when needed):

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
}

resource "aws_iam_role" "lambda_role" {
  name = "lambda-execution-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_lambda_function" "processor" {
  # Lambda needs the bucket to exist first
  depends_on = [aws_s3_bucket.data]
  
  function_name = "data-processor"
  role         = aws_iam_role.lambda_role.arn
  # ...
}
```

---

## 4. Terraform State

### What is State?

The **state file** (`terraform.tfstate`) is Terraform's source of truth about your infrastructure.

```
┌─────────────────────────────────────────────────────────┐
│              WHY DO WE NEED STATE?                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│ Without State:                                          │
│ - Terraform doesn't know what it created                │
│ - Can't detect drift                                    │
│ - Can't update existing resources                       │
│ - Would try to recreate everything                      │
│                                                          │
│ With State:                                             │
│ - Tracks all managed resources                          │
│ - Maps real-world objects to your config                │
│ - Improves performance (cached data)                    │
│ - Detects changes outside Terraform                     │
│ - Enables collaboration                                 │
└─────────────────────────────────────────────────────────┘
```

### State File Structure

```json
{
  "version": 4,
  "terraform_version": "1.5.0",
  "serial": 42,
  "lineage": "abc123-def456-ghi789",
  "outputs": {...},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web_server",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "ami": "ami-0c55b159cbfafe1f0",
            "id": "i-0abc123def456789",
            "instance_type": "t2.micro",
            "public_ip": "54.123.45.67",
            "tags": {
              "Name": "web-server-1"
            }
            // ... all other attributes
          }
        }
      ]
    }
    // ... more resources
  ]
}
```

### Local vs Remote State

#### Local State (Development Only)

```bash
# State stored locally
./terraform.tfstate
./terraform.tfstate.backup
./.terraform/
```

⚠️ **Problems with Local State:**
- ❌ No collaboration (single user only)
- ❌ Risk of losing state file
- ❌ No locking (concurrent applies can corrupt state)
- ❌ Credentials might be stored in plain text

#### Remote State (Production)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/infrastructure/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # For state locking
  }
}
```

**Benefits of Remote State:**
- ✅ Secure storage with encryption
- ✅ Team collaboration
- ✅ State locking prevents concurrent modifications
- ✅ Version history
- ✅ Automated backups

### Supported Backends

| Backend | Use Case |
|---------|----------|
| `local` | Development, learning |
| `s3` + DynamoDB | AWS environments |
| `azurerm` | Azure environments |
| `gcs` | Google Cloud environments |
| `consul` | On-premises, HashiCorp stack |
| `pg` | PostgreSQL backend |
| `http` | Custom HTTP endpoint |
| `remote` | Terraform Cloud/Enterprise |

### State Commands

```bash
# View current state
terraform state list

# Show specific resource
terraform state show aws_instance.web_server

# Move resource to different address
terraform state mv aws_instance.old aws_instance.new

# Remove resource from state (doesn't delete actual resource)
terraform state rm aws_instance.unwanted

# Import existing resource into state
terraform import aws_instance.existing i-1234567890abcdef0

# Force unlock state (if stuck)
terraform force-unlock LOCK_ID
```

### State Locking

When using remote backends, Terraform automatically locks state during operations:

```
Acquiring state lock...
Lock ID: abc123-def456-ghi789
Operation: terraform apply
Who: user@example.com
When: 2024-01-15 10:30:00 UTC

# If someone else tries to apply:
Error: State locked by another process

# To manually unlock (careful!):
terraform force-unlock abc123-def456-ghi789
```

---

## 5. Data Sources

### What are Data Sources?

**Data sources** allow Terraform to read information from existing infrastructure without managing it.

**Use Cases:**
- Look up AMI IDs
- Find existing VPCs or subnets
- Read secrets from AWS Secrets Manager
- Query DNS records
- Get current user/account info

### Syntax

```hcl
data "<TYPE>" "<LOCAL_NAME>" {
  <FILTER> = <VALUE>
  ...
}
```

### Common Examples

#### Lookup Latest AMI

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id  # Use looked-up AMI
  instance_type = "t2.micro"
}
```

#### Find Existing VPC

```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
  
  filter {
    name   = "availability-zone"
    values = ["us-east-1a"]
  }
}

resource "aws_instance" "in_default_vpc" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  subnet_id     = data.aws_subnets.public.ids[0]
}
```

#### Get Current Account Info

```hcl
data "aws_caller_identity" "current" {}

data "aws_region" "current" {}

output "account_info" {
  value = {
    account_id = data.aws_caller_identity.current.account_id
    arn        = data.aws_caller_identity.current.arn
    user_id    = data.aws_caller_identity.current.user_id
    region     = data.aws_region.current.name
  }
}
```

#### Read from Secrets Manager

```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/password"
}

resource "aws_db_instance" "main" {
  # ...
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

---

## 6. The Terraform Workflow

### Core Commands

```
┌─────────────────────────────────────────────────────────┐
│            TERRAFORM WORKFLOW                            │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. terraform init                                       │
│     ↓ Initialize working directory                       │
│     ↓ Download providers                                 │
│     ↓ Configure backend                                  │
│                                                          │
│  2. terraform plan                                       │
│     ↓ Refresh state                                      │
│     ↓ Compare config vs state                            │
│     ↓ Show execution plan                                │
│                                                          │
│  3. terraform apply                                      │
│     ↓ Execute planned changes                            │
│     ↓ Update state                                       │
│     ↓ Output results                                     │
│                                                          │
│  4. terraform destroy                                    │
│     ↓ Remove all managed resources                       │
│     ↓ Clean up state                                     │
└─────────────────────────────────────────────────────────┘
```

### Command Details

#### `terraform init`

```bash
# Initialize directory
terraform init

# With options
terraform init \
  -backend-config="bucket=my-bucket" \
  -backend-config="key=prod/terraform.tfstate" \
  -upgrade  # Upgrade providers
  -reconfigure  # Reconfigure backend
```

**What it does:**
- ✅ Initializes backend
- ✅ Downloads provider plugins
- ✅ Sets up `.terraform/` directory
- ✅ Validates basic syntax

#### `terraform plan`

```bash
# Create execution plan
terraform plan

# Save plan to file
terraform plan -out=tfplan

# Show detailed diff
terraform plan -detailed

# Target specific resources
terraform plan -target=aws_instance.web
```

**Plan Output Symbols:**
- `+` Create
- `-` Destroy
- `~` Update
- `-/+` Destroy and recreate

#### `terraform apply`

```bash
# Apply changes (will prompt for confirmation)
terraform apply

# Apply saved plan
terraform apply tfplan

# Auto-approve (use carefully!)
terraform apply -auto-approve

# Target specific resources
terraform apply -target=aws_instance.web

# Replace specific instance
terraform apply -replace=aws_instance.web
```

#### `terraform destroy`

```bash
# Destroy all resources
terraform destroy

# Auto-approve (DANGEROUS!)
terraform destroy -auto-approve

# Target specific resources to destroy
terraform destroy -target=aws_instance.web

# Keep specific resources
terraform destroy -exclude=aws_s3_bucket.important
```

### Complete Workflow Example

```bash
# Step 1: Initialize
$ terraform init
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.12.0...
- Installed hashicorp/aws v5.12.0
Terraform has been successfully initialized!

# Step 2: Write configuration
# (Create main.tf with your resources)

# Step 3: Plan
$ terraform plan
Refreshing Terraform state in-memory prior to plan...

Terraform will perform the following actions:

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + id               = (known after apply)
      + cidr_block       = "10.0.0.0/16"
      + instance_tenancy = "default"
      + tags             = {
          + "Name" = "main-vpc"
        }
    }

  # aws_subnet.public will be created
  + resource "aws_subnet" "public" {
      + id                = (known after apply)
      + vpc_id            = (known after apply)
      + cidr_block        = "10.0.1.0/24"
      + availability_zone = "us-east-1a"
    }

Plan: 2 to add, 0 to change, 0 to destroy.

# Step 4: Apply
$ terraform apply
# Review the plan and type 'yes' to confirm
aws_vpc.main: Creating...
aws_vpc.main: Creation complete after 3s
aws_subnet.public: Creating...
aws_subnet.public: Creation complete after 2s

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

# Step 5: Verify
$ aws ec2 describe-vpcs
# See your new VPC!

# Step 6: Make changes
# Edit main.tf, change cidr_block

$ terraform plan
# Shows ~ update for aws_vpc.main

$ terraform apply
# Applies the change

# Step 7: Cleanup (when done)
$ terraform destroy
# Removes all resources
```

---

## 7. Complete Example: Simple Web Server

Let's put it all together!

### Directory Structure

```
web-server/
├── main.tf           # Main configuration
├── variables.tf      # Input variables
├── outputs.tf        # Output values
├── terraform.tfvars  # Variable values (optional)
└── README.md
```

### main.tf

```hcl
terraform {
  required_version = ">= 1.0.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "web-server/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true
  }
}

provider "aws" {
  region = var.aws_region
}

# Data source: Find latest Amazon Linux AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Allow HTTP and SSH traffic"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP from anywhere"
  }
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.ssh_cidr_block]
    description = "SSH from trusted IPs"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
  }
  
  tags = {
    Name        = "${var.project_name}-web-sg"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "${var.project_name}-vpc"
    Environment = var.environment
  }
}

# Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.subnet_cidr
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true
  
  tags = {
    Name        = "${var.project_name}-public-subnet"
    Environment = var.environment
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name        = "${var.project_name}-igw"
    Environment = var.environment
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
    Name        = "${var.project_name}-public-rt"
    Environment = var.environment
  }
}

# Route Table Association
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name              = var.key_name
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Welcome to ${var.project_name}</h1>" > /var/www/html/index.html
              echo "<p>Server: $(hostname)</p>" >> /var/www/html/index.html
              EOF
  
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 20
    encrypted             = true
    delete_on_termination = true
  }
  
  tags = {
    Name        = "${var.project_name}-web-server"
    Environment = var.environment
    Project     = var.project_name
  }
}
```

### variables.tf

```hcl
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "my-web-app"
}

variable "environment" {
  description = "Environment (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_cidr" {
  description = "CIDR block for subnet"
  type        = string
  default     = "10.0.1.0/24"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "key_name" {
  description = "EC2 key pair name for SSH access"
  type        = string
}

variable "ssh_cidr_block" {
  description = "CIDR block allowed for SSH access"
  type        = string
  default     = "0.0.0.0/0"
}
```

### outputs.tf

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "public_ip" {
  description = "Public IP address of the web server"
  value       = aws_instance.web.public_ip
}

output "public_dns" {
  description = "Public DNS name of the web server"
  value       = aws_instance.web.public_dns
}

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "security_group_id" {
  description = "ID of the security group"
  value       = aws_security_group.web.id
}
```

### terraform.tfvars (Optional)

```hcl
project_name   = "production-webapp"
environment    = "production"
aws_region     = "us-east-1"
instance_type  = "t3.small"
key_name       = "production-key"
ssh_cidr_block = "203.0.113.0/24"  # Your office IP
```

### Deploy It!

```bash
# Initialize
terraform init

# Plan with variable overrides
terraform plan \
  -var="project_name=demo-app" \
  -var="environment=staging"

# Or use tfvars file
terraform plan -var-file=terraform.tfvars

# Apply
terraform apply -var-file=terraform.tfvars

# Access your server
curl http://$(terraform output -raw public_ip)
```

---

## Summary

### Key Takeaways

✅ **Providers** enable Terraform to work with different platforms
✅ **Resources** are the building blocks of your infrastructure
✅ **State** tracks your infrastructure's current state
✅ **Data Sources** read information from existing resources
✅ **Workflow**: `init` → `plan` → `apply` → `destroy`
✅ Always use **remote state** for team collaboration
✅ Understand **dependencies** between resources

### Best Practices

1. **Version pin providers** to avoid breaking changes
2. **Use remote state** with locking for teams
3. **Tag all resources** for cost tracking and organization
4. **Use variables** for environment-specific values
5. **Review plans** before applying
6. **Store state securely** (encrypt at rest)
7. **Backup state regularly**
8. **Use workspaces** for multiple environments

### What's Next?

➡️ **Next Section**: [Variables & Outputs](./03-terraform-variables-outputs.md)

Learn about:
- Input variables and validation
- Output values
- Local values
- Expressions and functions
- Dynamic blocks

---

## Additional Resources

- [Terraform Registry](https://registry.terraform.io/)
- [Provider Documentation](https://registry.terraform.io/browse/providers)
- [Terraform State Documentation](https://www.terraform.io/docs/state/)
- [AWS Provider Guide](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform CLI Commands](https://www.terraform.io/docs/cli/commands/index.html)

---

**Ready to continue?** Move to [Section 03: Variables & Outputs](./03-terraform-variables-outputs.md)
