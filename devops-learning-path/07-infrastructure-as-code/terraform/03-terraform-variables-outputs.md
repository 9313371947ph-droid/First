# 03. Terraform Variables & Outputs

## 🎯 Learning Objectives
By the end of this section, you will be able to:
- Define and use variables in Terraform configurations
- Create different variable types (string, number, bool, list, map, object)
- Implement input validation with constraints
- Use outputs to expose resource information
- Organize variables in separate files
- Pass variables via CLI, environment variables, and .tfvars files

## 📋 Table of Contents
1. Why Variables Matter
2. Variable Types
3. Defining Variables
4. Variable Validation
5. Using Variables
6. Output Values
7. Variable Precedence
8. Best Practices
9. Hands-on Lab

---

## 1. Why Variables Matter

Variables make your Terraform configurations:
- **Reusable**: Same code for multiple environments
- **Flexible**: Easy to change values without modifying code
- **Maintainable**: Centralized configuration management
- **Secure**: Separate sensitive data from logic
- **Collaborative**: Team members can customize deployments

### Without Variables
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "web-server-prod-us-east-1"
    Environment = "production"
    Region = "us-east-1"
  }
}
```

### With Variables
```hcl
variable "environment" {
  type    = string
  default = "production"
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "region" {
  type    = string
  default = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  
  tags = {
    Name        = "web-server-${var.environment}-${var.region}"
    Environment = var.environment
    Region      = var.region
  }
}
```

---

## 2. Variable Types

Terraform supports several variable types:

### Primitive Types

#### String
```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}
```

#### Number
```hcl
variable "instance_count" {
  type    = number
  default = 2
}
```

#### Bool
```hcl
variable "enable_monitoring" {
  type    = bool
  default = true
}
```

### Complex Types

#### List
```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "allowed_ports" {
  type    = list(number)
  default = [80, 443, 22]
}
```

#### Map
```hcl
variable "instance_tags" {
  type = map(string)
  default = {
    Project     = "myapp"
    Owner       = "devops-team"
    CostCenter  = "engineering"
  }
}
```

#### Object
```hcl
variable "database_config" {
  type = object({
    engine         = string
    version        = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  
  default = {
    engine         = "mysql"
    version        = "8.0"
    instance_class = "db.t3.medium"
    storage_gb     = 100
    multi_az       = true
  }
}
```

#### Tuple (Fixed-length list with mixed types)
```hcl
variable "coordinates" {
  type    = tuple([number, number])
  default = [40.7128, -74.0060]  # NYC coordinates
}
```

#### Any (No type constraint)
```hcl
variable "flexible_value" {
  type    = any
  default = "can be anything"
}
```

---

## 3. Defining Variables

### Basic Variable Definition
```hcl
variable "instance_name" {
  description = "Name tag for EC2 instances"
  type        = string
  default     = "web-server"
}
```

### Variable with No Default (Required)
```hcl
variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  # No default = required
}
```

### Variable with Sensitive Data
```hcl
variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true  # Hides value in logs/output
}
```

### Variable with Multiple Constraints
```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  
  validation {
    condition     = can(regex("^t[23]\\.(micro|small|medium|large)$", var.instance_type))
    error_message = "Instance type must be t2 or t3 series (micro, small, medium, or large)."
  }
}
```

---

## 4. Variable Validation

Terraform 0.13+ supports validation blocks:

### Length Validation
```hcl
variable "password" {
  description = "Database password"
  type        = string
  sensitive   = true
  
  validation {
    condition     = length(var.password) >= 16
    error_message = "Password must be at least 16 characters long."
  }
  
  validation {
    condition     = can(regex("[A-Z]", var.password))
    error_message = "Password must contain at least one uppercase letter."
  }
  
  validation {
    condition     = can(regex("[0-9]", var.password))
    error_message = "Password must contain at least one number."
  }
}
```

### Range Validation
```hcl
variable "replica_count" {
  description = "Number of database read replicas"
  type        = number
  
  validation {
    condition     = var.replica_count >= 0 && var.replica_count <= 5
    error_message = "Replica count must be between 0 and 5."
  }
}
```

### Regex Pattern Validation
```hcl
variable "email" {
  description = "Admin email address"
  type        = string
  
  validation {
    condition     = can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", var.email))
    error_message = "Must be a valid email address."
  }
}
```

### Custom Validation Functions
```hcl
variable "cidr_block" {
  description = "VPC CIDR block"
  type        = string
  
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block notation."
  }
}
```

---

## 5. Using Variables

### Referencing Variables
```hcl
# In resource arguments
resource "aws_instance" "web" {
  instance_type = var.instance_type
  ami           = var.ami_id
  
  tags = {
    Name = var.instance_name
  }
}

# In locals
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
  }
}

# In data sources
data "aws_ami" "ubuntu" {
  most_recent = true
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  
  owners = [var.ami_owner]  # Variable in data source
}

# In outputs
output "instance_id" {
  value = aws_instance.web.id
}

output "instance_public_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of the web server"
}
```

### Using Complex Variables
```hcl
# List iteration
resource "aws_security_group_rule" "ingress" {
  for_each = toset(var.allowed_ports)
  
  security_group_id = aws_security_group.web.id
  type              = "ingress"
  from_port         = each.value
  to_port           = each.value
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}

# Map iteration
resource "aws_instance" "servers" {
  for_each = var.server_configs
  
  instance_type = each.value.instance_type
  ami           = each.value.ami
  subnet_id     = each.value.subnet_id
  
  tags = {
    Name = each.key
    Role = each.value.role
  }
}

# Object attributes
resource "aws_db_instance" "main" {
  engine               = var.database_config.engine
  engine_version       = var.database_config.version
  instance_class       = var.database_config.instance_class
  allocated_storage    = var.database_config.storage_gb
  multi_az             = var.database_config.multi_az
}
```

---

## 6. Output Values

Outputs display information after `terraform apply` and can be used by other modules.

### Basic Output
```hcl
output "instance_id" {
  description = "ID of the created EC2 instance"
  value       = aws_instance.web.id
}
```

### Output with Formatting
```hcl
output "connection_info" {
  description = "Connection details for the web server"
  value = {
    public_ip   = aws_instance.web.public_ip
    public_dns  = aws_instance.web.public_dns
    ssh_command = "ssh ec2-user@${aws_instance.web.public_ip}"
  }
}
```

### Sensitive Output
```hcl
output "database_password" {
  description = "Master password for the database"
  value       = aws_db_instance.main.master_password
  sensitive   = true  # Hides value in CLI output
}
```

### Conditional Output
```hcl
output "nat_gateway_ip" {
  description = "Public IP of NAT Gateway"
  value       = var.create_nat ? aws_nat_gateway.main[0].public_ip : null
}
```

### Using Outputs from Other Modules
```hcl
module "vpc" {
  source = "./modules/vpc"
  
  cidr_block = "10.0.0.0/16"
}

module "instances" {
  source = "./modules/instances"
  
  vpc_id     = module.vpc.vpc_id      # Reference output
  subnet_ids = module.vpc.private_subnet_ids
}
```

---

## 7. Variable Precedence

Terraform loads variables from multiple sources with this precedence (highest to lowest):

1. **CLI flags**: `-var` or `-var-file`
2. **Environment variables**: `TF_VAR_<name>`
3. **terraform.tfvars**: Auto-loaded if present
4. ***.auto.tfvars**: Files matching pattern (alphabetical order)
5. **Variable defaults**: Defined in .tf files

### Example Setup

#### variables.tf
```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}
```

#### terraform.tfvars
```hcl
region        = "us-west-2"
instance_type = "t3.medium"
```

#### dev.auto.tfvars
```hcl
instance_type = "t3.small"
environment   = "development"
```

#### prod.auto.tfvars
```hcl
instance_type = "t3.large"
environment   = "production"
```

#### Setting via Environment
```bash
export TF_VAR_region="eu-west-1"
export TF_VAR_instance_type="t3.xlarge"
terraform plan
```

#### Setting via CLI
```bash
# Single variable
terraform apply -var "region=ap-south-1"

# Multiple variables
terraform apply -var "region=ap-south-1" -var "instance_type=t3.2xlarge"

# From file
terraform apply -var-file="production.tfvars"
```

---

## 8. Best Practices

### ✅ DO

#### Separate Variables File
```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars
```

#### Use Descriptive Names
```hcl
# Good
variable "database_instance_class" {
  description = "RDS instance class for the primary database"
  type        = string
}

# Bad
variable "db_size" {
  type = string
}
```

#### Validate Early
```hcl
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

#### Mark Sensitive Data
```hcl
variable "api_key" {
  type      = string
  sensitive = true
}

variable "db_password" {
  type      = string
  sensitive = true
}
```

#### Use Type Constraints
```hcl
# Good - explicit type
variable "max_connections" {
  type    = number
  default = 100
}

# Bad - no type
variable "max_connections" {
  default = 100
}
```

### ❌ DON'T

#### Hardcode Values in Resources
```hcl
# Bad
resource "aws_instance" "web" {
  instance_type = "t2.micro"  # Hardcoded!
}

# Good
resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

#### Store Secrets in Version Control
```hcl
# Bad - committed to git
variable "password" {
  default = "SuperSecret123!"
}

# Good - use environment or secret manager
variable "password" {
  type      = string
  sensitive = true
  # No default, pass via TF_VAR_password or secret manager
}
```

#### Overuse 'any' Type
```hcl
# Bad
variable "config" {
  type = any
}

# Good
variable "config" {
  type = object({
    region   = string
    vpc_cidr = string
    subnets  = list(string)
  })
}
```

---

## 9. Hands-on Lab

### Lab Objective
Create a multi-environment variable configuration for a web application deployment.

### Step 1: Create Project Structure
```bash
mkdir -p terraform-variables-lab/{modules,environments}
cd terraform-variables-lab
```

### Step 2: Define Variables (variables.tf)
```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

variable "allowed_ssh_cidrs" {
  description = "CIDR blocks allowed for SSH access"
  type        = list(string)
  default     = []
}

variable "tags" {
  description = "Additional tags for resources"
  type        = map(string)
  default     = {}
}

variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
  
  validation {
    condition     = length(var.db_password) >= 16
    error_message = "Password must be at least 16 characters."
  }
}
```

### Step 3: Create Main Configuration (main.tf)
```hcl
provider "aws" {
  region = var.region
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_security_group" "web" {
  name        = "web-sg-${var.environment}"
  description = "Security group for web servers"
  
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
  
  dynamic "ingress" {
    for_each = var.allowed_ssh_cidrs
    content {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = [ingress.value]
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(
    {
      Name        = "web-sg-${var.environment}"
      Environment = var.environment
    },
    var.tags
  )
}

resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  
  monitoring = var.enable_monitoring
  
  vpc_security_group_ids = [aws_security_group.web.id]
  
  tags = merge(
    {
      Name        = "web-${var.environment}-${count.index + 1}"
      Environment = var.environment
      ManagedBy   = "terraform"
    },
    var.tags
  )
}
```

### Step 4: Create Outputs (outputs.tf)
```hcl
output "instance_ids" {
  description = "IDs of created EC2 instances"
  value       = aws_instance.web[*].id
}

output "instance_public_ips" {
  description = "Public IPs of created EC2 instances"
  value       = aws_instance.web[*].public_ip
}

output "security_group_id" {
  description = "ID of the security group"
  value       = aws_security_group.web.id
}

output "environment_summary" {
  description = "Summary of deployed environment"
  value = {
    environment    = var.environment
    region         = var.region
    instance_type  = var.instance_type
    instance_count = var.instance_count
    monitoring     = var.enable_monitoring
  }
}
```

### Step 5: Create Environment Files

#### environments/dev.tfvars
```hcl
environment       = "dev"
region            = "us-east-1"
instance_type     = "t2.micro"
instance_count    = 1
enable_monitoring = false
allowed_ssh_cidrs = ["10.0.0.0/8"]

tags = {
  Project = "WebApp"
  Team    = "Development"
}
```

#### environments/staging.tfvars
```hcl
environment       = "staging"
region            = "us-east-1"
instance_type     = "t3.small"
instance_count    = 2
enable_monitoring = true
allowed_ssh_cidrs = ["10.0.0.0/8", "192.168.1.0/24"]

tags = {
  Project = "WebApp"
  Team    = "QA"
}
```

#### environments/prod.tfvars
```hcl
environment       = "prod"
region            = "us-east-1"
instance_type     = "t3.medium"
instance_count    = 3
enable_monitoring = true
allowed_ssh_cidrs = ["10.0.0.0/8"]  # Office IP only

tags = {
  Project = "WebApp"
  Team    = "Production"
  CostCenter = "PROD-001"
}
```

### Step 6: Deploy Different Environments

```bash
# Initialize
terraform init

# Deploy development environment
terraform plan -var-file=environments/dev.tfvars
terraform apply -var-file=environments/dev.tfvars

# Deploy staging environment (in different workspace or directory)
terraform plan -var-file=environments/staging.tfvars
terraform apply -var-file=environments/staging.tfvars

# Deploy production with override
terraform apply \
  -var-file=environments/prod.tfvars \
  -var "instance_count=5" \
  -var "db_password=$(openssl rand -base64 24)"
```

### Step 7: Test Variable Precedence

```bash
# Default from variables.tf
terraform plan

# Override with environment variable
export TF_VAR_instance_type="t3.large"
terraform plan

# Override with CLI flag (highest precedence)
terraform plan -var "instance_type=t3.xlarge"
```

### Cleanup
```bash
terraform destroy -var-file=environments/dev.tfvars -auto-approve
```

---

## 🔑 Key Takeaways

1. **Variables enable reusability** - Write once, deploy everywhere
2. **Use strong typing** - Catch errors early with type constraints
3. **Validate inputs** - Prevent invalid configurations before deployment
4. **Mark sensitive data** - Protect secrets in logs and outputs
5. **Organize by environment** - Separate tfvars files for dev/staging/prod
6. **Understand precedence** - Know which variable source wins
7. **Document everything** - Clear descriptions help team collaboration

---

## 📚 Additional Resources

- [Terraform Variables Documentation](https://www.terraform.io/docs/language/values/variables.html)
- [Terraform Outputs Documentation](https://www.terraform.io/docs/language/values/outputs.html)
- [Input Variable Design Patterns](https://www.terraform.io/docs/language/values/variables.html#input-variable-design)
- [Terraform Registry - AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

---

**Next**: [04-terraform-state-management.md](04-terraform-state-management.md) - Learn how Terraform tracks your infrastructure state
