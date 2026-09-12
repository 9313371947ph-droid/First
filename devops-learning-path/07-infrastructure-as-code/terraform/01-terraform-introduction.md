# Terraform Section 01: Introduction to Infrastructure as Code

## 🎯 Learning Objectives
- Understand what Infrastructure as Code (IaC) is and why it matters
- Compare traditional vs. IaC approaches
- Learn why Terraform became the industry standard
- Understand Terraform's core philosophy and advantages
- Set up your development environment

---

## 1. What is Infrastructure as Code (IaC)?

### Definition
**Infrastructure as Code (IaC)** is the practice of managing and provisioning infrastructure through machine-readable definition files, rather than physical hardware configuration or interactive configuration tools.

### The Traditional Way (Before IaC)
```
┌─────────────────────────────────────────────────────────┐
│              MANUAL INFRASTRUCTURE MANAGEMENT            │
├─────────────────────────────────────────────────────────┤
│ 1. Request server from IT team                          │
│ 2. Wait days/weeks for provisioning                     │
│ 3. Manually install OS, packages, configurations        │
│ 4. Document changes in Wiki/Excel (if lucky)            │
│ 5. "Snowflake servers" - each one unique                │
│ 6. Disaster recovery = panic and recreate manually      │
│ 7. Environment drift: Dev ≠ Staging ≠ Production        │
└─────────────────────────────────────────────────────────┘
```

**Problems with Manual Approach:**
- ❌ **Slow**: Days or weeks to provision resources
- ❌ **Error-prone**: Human mistakes in configuration
- ❌ **Inconsistent**: Each environment slightly different
- ❌ **Not reproducible**: Hard to recreate exact setup
- ❌ **No version control**: Can't track who changed what
- ❌ **Difficult collaboration**: Knowledge silos
- ❌ **Expensive**: High operational overhead

### The IaC Way
```
┌─────────────────────────────────────────────────────────┐
│              INFRASTRUCTURE AS CODE                      │
├─────────────────────────────────────────────────────────┤
│ 1. Write code defining desired infrastructure           │
│ 2. Version control in Git                               │
│ 3. Review changes via Pull Requests                     │
│ 4. Apply changes automatically                          │
│ 5. Identical environments every time                    │
│ 6. Destroy and recreate in minutes                      │
│ 7. Complete audit trail of all changes                  │
└─────────────────────────────────────────────────────────┘
```

**Benefits of IaC:**
- ✅ **Speed**: Provision infrastructure in minutes
- ✅ **Consistency**: Identical environments every time
- ✅ **Reproducibility**: Recreate entire infrastructure from scratch
- ✅ **Version Control**: Track all changes in Git
- ✅ **Collaboration**: Team reviews and approves changes
- ✅ **Documentation**: Code IS the documentation
- ✅ **Cost Reduction**: Automate repetitive tasks
- ✅ **Disaster Recovery**: Rebuild quickly from code

---

## 2. Types of IaC Tools

### Imperative vs. Declarative

#### **Imperative Approach** (How to do it)
- You specify **exact steps** to achieve the desired state
- Examples: Shell scripts, Ansible (to some extent), Chef
- Focus on the **process**

```bash
# Imperative example (Shell script)
#!/bin/bash
aws ec2 run-instances --image-id ami-12345 --instance-type t2.micro
aws ec2 create-tags --resources i-12345 --tags Key=Name,Value=WebServer
aws ec2 authorize-security-group-ingress --group-id sg-123 --port 80
```
**Problem**: What if you run this twice? You get two servers!

#### **Declarative Approach** (What you want)
- You specify **desired end state**
- Tool figures out how to achieve it
- Examples: Terraform, CloudFormation, Pulumi
- Focus on the **result**

```hcl
# Declarative example (Terraform)
resource "aws_instance" "web_server" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
  
  vpc_security_group_ids = ["sg-123"]
}
```
**Advantage**: Run this 100 times → still only ONE server. Terraform knows current state vs. desired state.

### Comparison Table

| Aspect | Imperative | Declarative |
|--------|-----------|-------------|
| **Focus** | How to achieve | What to achieve |
| **Idempotency** | Must be coded manually | Built-in |
| **State Management** | None (or manual) | Automatic |
| **Learning Curve** | Easier initially | Steeper but more powerful |
| **Best For** | Simple tasks, ad-hoc | Complex infrastructure, production |
| **Examples** | Shell scripts, Ansible | Terraform, CloudFormation |

---

## 3. Why Terraform?

### Overview
**Terraform** by HashiCorp is an open-source Infrastructure as Code software tool that provides a consistent CLI workflow to manage hundreds of cloud services.

### Key Features

#### 1. **Multi-Cloud Support**
Terraform works with 100+ providers:
- ☁️ AWS, Azure, Google Cloud
- 🏢 VMware, OpenStack
- 🔧 Kubernetes, Docker
- 📊 Datadog, New Relic
- 🔐 Vault, Consul
- And many more...

```hcl
# Use multiple clouds in same configuration
provider "aws" {
  region = "us-east-1"
}

provider "azurerm" {
  features {}
}

provider "google" {
  project = "my-project"
  region  = "us-central1"
}
```

#### 2. **State Management**
Terraform maintains a **state file** (`terraform.tfstate`) that tracks:
- What resources exist
- Their current configuration
- Relationships between resources
- Metadata for performance

This enables:
- ✅ Detecting drift (manual changes)
- ✅ Planning changes before applying
- ✅ Efficient updates (only change what's needed)
- ✅ Dependency management

#### 3. **Execution Plans**
```bash
$ terraform plan
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-12345"
      + instance_type = "t2.micro"
      + tags          = {
          + "Name" = "WebServer"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```
**See exactly what will happen BEFORE making changes!**

#### 4. **Resource Graph**
Terraform builds a dependency graph:
- Determines correct order of operations
- Parallelizes non-dependent operations
- Automatically handles dependencies

```
Database ← Application Server ← Load Balancer
                ↑
          Security Group
```

#### 5. **Change Automation**
Complex changesets applied with minimal human interaction:
```bash
terraform init    # Initialize
terraform plan    # Review changes
terraform apply   # Execute
```

---

## 4. Terraform vs. Other Tools

### Terraform vs. Ansible
| Feature | Terraform | Ansible |
|---------|-----------|---------|
| **Primary Use** | Provisioning infrastructure | Configuration management |
| **Approach** | Declarative | Imperative/Declarative |
| **State** | Maintains state file | Stateless (mostly) |
| **Agents** | Agentless | Agentless |
| **Best For** | Creating resources | Configuring existing resources |
| **Example** | Create EC2, VPC, RDS | Install packages, start services |

**Modern Approach**: Use BOTH!
```
Terraform → Create infrastructure (servers, networks, DBs)
    ↓
Ansible → Configure servers (install apps, deploy code)
```

### Terraform vs. CloudFormation
| Feature | Terraform | CloudFormation |
|---------|-----------|----------------|
| **Vendor** | HashiCorp (multi-cloud) | AWS only |
| **Language** | HCL (human-friendly) | JSON/YAML |
| **State Management** | Yes (flexible backends) | Yes (AWS managed) |
| **Multi-cloud** | ✅ 100+ providers | ❌ AWS only |
| **Community** | Large, active | AWS-focused |
| **Learning Curve** | Moderate | Moderate |

### Terraform vs. Pulumi
| Feature | Terraform | Pulumi |
|---------|-----------|--------|
| **Language** | HCL (domain-specific) | Python, Go, Node.js, C# |
| **State** | File-based | Managed service option |
| **Adoption** | Industry standard | Growing |
| **Ecosystem** | Mature, extensive | Smaller but growing |

---

## 5. Terraform Workflow (The Core Loop)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   WRITE     │────▶│    PLAN     │────▶│    APPLY    │
│  Code (.tf) │     │  Preview    │     │  Execute    │
└─────────────┘     └─────────────┘     └─────────────┘
       ▲                                      │
       │                                      ▼
       │                            ┌─────────────┐
       └────────────────────────────│   STATE     │
                                    │  (tfstate)  │
                                    └─────────────┘
```

### Step 1: Write
Create `.tf` files defining your infrastructure:
```hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

### Step 2: Init
Initialize your working directory:
```bash
terraform init
```
- Downloads provider plugins
- Sets up backend
- Prepares working directory

### Step 3: Plan
Preview changes before applying:
```bash
terraform plan
```
- Compares desired state (code) with current state
- Shows what will be created, changed, or destroyed
- **No actual changes made yet!**

### Step 4: Apply
Execute the planned changes:
```bash
terraform apply
```
- Creates/updates/deletes resources
- Updates state file
- Makes real changes to infrastructure

### Step 5: Destroy (when needed)
Tear down infrastructure:
```bash
terraform destroy
```
- Removes all resources managed by Terraform
- Useful for cleanup, cost savings, testing

---

## 6. Real-World Example: Before and After

### Scenario: Deploy a Web Application

#### Before Terraform (Manual Process)
```
Day 1: 
- Request VPC setup from network team (wait 2 days)
- Create security groups manually (forget one rule)
- Launch EC2 instances via AWS Console (typo in instance type)
- Install nginx manually (different versions on each server)
- Configure load balancer (miss health check settings)

Day 3:
- Notice issues, try to fix manually
- Can't remember exact changes made
- Production different from staging

Week 2:
- Need to replicate for disaster recovery
- Start from scratch, make different mistakes
- Environments drift further apart
```

#### After Terraform (IaC Process)
```
Day 1 Morning:
- Write Terraform code (VPC, subnets, security groups, EC2, ALB)
- Commit to Git
- Create Pull Request

Day 1 Afternoon:
- Team reviews PR, suggests improvements
- Approve and merge
- Run: terraform plan (review changes)
- Run: terraform apply

Day 1 Evening:
- Infrastructure fully deployed
- Exact replica ready for staging
- Complete documentation in code

Day 2:
- Need to scale? Change instance count in code
- Need different region? Copy code, change region variable
- Disaster recovery? Run same code in different region

Result:
✅ Consistent environments
✅ Fast deployment (hours → minutes)
✅ Easy scaling and replication
✅ Complete audit trail
✅ Team collaboration via Git
```

---

## 7. Installation and Setup

### Prerequisites
- Operating System: Linux, macOS, or Windows
- Internet connection (to download providers)
- Cloud account (AWS Free Tier recommended)

### Install Terraform

#### macOS (Homebrew)
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

#### Linux (Ubuntu/Debian)
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

#### Windows (Chocolatey)
```powershell
choco install terraform
```

#### Verify Installation
```bash
terraform version
```
Output:
```
Terraform v1.6.0
on linux_amd64
```

### Install AWS CLI (for AWS examples)
```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure credentials
aws configure
```

### Configure AWS Credentials
```bash
aws configure
# Enter:
# AWS Access Key ID: [your-key]
# AWS Secret Access Key: [your-secret]
# Default region name: us-east-1
# Default output format: json
```

**⚠️ Security Note**: Never commit AWS credentials to Git! Use environment variables or IAM roles in production.

---

## 8. Your First Terraform Configuration

### Directory Structure
```
my-first-terraform/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars (optional)
```

### main.tf
```hcl
# Define the provider
provider "aws" {
  region = "us-east-1"
}

# Create a VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name        = "main-vpc"
    Environment = "learning"
  }
}

# Create a subnet
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  
  tags = {
    Name = "public-subnet"
  }
}

# Create a security group
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTP traffic"
  vpc_id      = aws_vpc.main.id

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

# Create an EC2 instance
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id
  
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from Terraform!</h1>" > /var/www/html/index.html
              EOF

  tags = {
    Name        = "web-server"
    Environment = "learning"
  }
}
```

### variables.tf
```hcl
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "learning"
}
```

### outputs.tf
```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "instance_public_ip" {
  description = "Public IP of the web server"
  value       = aws_instance.web_server.public_ip
}

output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web_server.id
}
```

### Execute Your First Infrastructure
```bash
# Navigate to directory
cd my-first-terraform

# Initialize Terraform
terraform init

# Preview changes
terraform plan

# Apply changes (type 'yes' when prompted)
terraform apply

# View outputs
echo "VPC ID: $(terraform output vpc_id)"
echo "Web Server IP: $(terraform output instance_public_ip)"

# Visit in browser
open http://$(terraform output -raw instance_public_ip)

# Cleanup (when done learning)
terraform destroy
```

---

## 9. Key Terminology

| Term | Definition |
|------|------------|
| **Provider** | Plugin that interacts with APIs (AWS, Azure, etc.) |
| **Resource** | Basic building block (EC2, S3, VPC, etc.) |
| **State** | Terraform's database of managed resources |
| **Module** | Reusable container for multiple resources |
| **Variable** | Input parameter for customization |
| **Output** | Value exported after deployment |
| **Backend** | Where state file is stored (local, S3, etc.) |
| **Workspace** | Separate state files for different environments |
| **Provisioner** | Execute scripts on local/remote machines |

---

## 10. Best Practices (Introduction)

### ✅ DO:
- Use version control (Git) for all Terraform code
- Enable remote state storage (S3 + DynamoDB)
- Use variables for environment-specific values
- Tag all resources consistently
- Review `terraform plan` before every `apply`
- Use modules for reusability
- Implement state locking

### ❌ DON'T:
- Commit sensitive data (passwords, keys) to Git
- Share state files publicly
- Make manual changes to Terraform-managed resources
- Use latest version pinning (`version = "latest"`)
- Ignore state file conflicts
- Run apply without reviewing plan

---

## 11. Knowledge Check

### Quiz Questions

**Q1**: What is the main advantage of declarative over imperative IaC?
<details>
<summary>Click for Answer</summary>
Declarative approach focuses on the desired end state, not the steps. This provides built-in idempotency - running the same code multiple times produces the same result without duplicates.
</details>

**Q2**: What does `terraform plan` do?
<details>
<summary>Click for Answer</summary>
It compares the desired state (code) with the current state (tfstate) and shows what changes would be made WITHOUT actually executing them. It's a dry-run preview.
</details>

**Q3**: Why is state management important in Terraform?
<details>
<summary>Click for Answer</summary>
State allows Terraform to track what resources exist, their configuration, and relationships. Without state, Terraform couldn't detect drift, plan efficient updates, or manage dependencies.
</details>

**Q4**: When should you use Terraform vs. Ansible?
<details>
<summary>Click for Answer</summary>
Use Terraform for provisioning infrastructure (creating resources like VMs, networks, databases). Use Ansible for configuring those resources (installing software, deploying applications). They complement each other.
</details>

**Q5**: What happens if you manually change a Terraform-managed resource?
<details>
<summary>Click for Answer</summary>
On next `terraform plan`, Terraform will detect the drift between the state file and actual infrastructure, and propose changes to bring it back to the desired state defined in code.
</details>

---

## 12. Hands-On Exercise

### Task: Deploy and Modify Infrastructure

1. **Setup**
   ```bash
   mkdir ~/terraform-learning
   cd ~/terraform-learning
   ```

2. **Create Basic Infrastructure**
   - Copy the example code from Section 8
   - Run `terraform init`
   - Run `terraform plan`
   - Run `terraform apply`

3. **Observe State**
   ```bash
   cat terraform.tfstate
   # Find your EC2 instance details
   ```

4. **Make a Change**
   - Edit `main.tf`: Change instance type from `t2.micro` to `t2.small`
   - Run `terraform plan` (see what changed)
   - Run `terraform apply`

5. **Add a Resource**
   - Add an S3 bucket to your configuration:
   ```hcl
   resource "aws_s3_bucket" "my_bucket" {
     bucket = "unique-bucket-name-12345"
     
     tags = {
       Name        = "My Bucket"
       Environment = "learning"
     }
   }
   ```
   - Apply the change

6. **Destroy Everything**
   ```bash
   terraform destroy
   ```

---

## 13. Common Errors and Troubleshooting

### Error: "Provider not found"
```bash
Error: Could not find required provider
```
**Solution**: Run `terraform init` to download providers.

### Error: "State file locked"
```bash
Error: Failed to lock state file
```
**Solution**: 
- If using remote state, check if another process is running
- Use `terraform force-unlock <LOCK_ID>` (careful!)

### Error: "Credentials not found"
```bash
Error: No valid credential sources found
```
**Solution**: 
- Run `aws configure`
- Or set environment variables:
  ```bash
  export AWS_ACCESS_KEY_ID=your_key
  export AWS_SECRET_ACCESS_KEY=your_secret
  ```

### Error: "Resource already exists"
```bash
Error: ResourceAlreadyExistsException
```
**Solution**: Import existing resource into state:
```bash
terraform import aws_instance.example i-1234567890abcdef0
```

---

## 14. Next Steps

You've learned:
- ✅ What IaC is and why it matters
- ✅ Terraform's advantages and workflow
- ✅ How to install and configure Terraform
- ✅ Your first Terraform configuration

**Next Section**: [02-terraform-core-concepts.md](02-terraform-core-concepts.md)

In the next section, we'll dive deep into:
- Providers and how they work
- Resource types and arguments
- Understanding state files in detail
- Data sources vs. Resources
- Interpolation and expressions

---

## 15. Additional Resources

### Documentation
- [Terraform Official Docs](https://www.terraform.io/docs)
- [AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform Language Reference](https://www.terraform.io/docs/language)

### Practice Platforms
- [Katacoda Terraform Scenarios](https://www.katacoda.com/courses/terraform)
- [HashiCorp Learn](https://learn.hashicorp.com/terraform)

### Books
- "Terraform: Up & Running" by Yevgeniy Brikman
- "Infrastructure as Code" by Kief Morris

---

**🎉 Congratulations!** You've completed Section 01. Ready to dive deeper into Terraform's core concepts?
