# Module 07: Infrastructure as Code (IaC)

## 🎯 Learning Objectives
By the end of this module, you will be able to:
- Understand the philosophy and benefits of Infrastructure as Code
- Provision cloud infrastructure using Terraform
- Configure and manage servers using Ansible
- Implement state management and remote backends
- Create reusable modules and roles
- Secure credentials with Terraform variables and Ansible Vault
- Build complete CI/CD pipelines for infrastructure deployment

## 📚 Module Structure

### Part A: Terraform (Infrastructure Provisioning)
1. **01-terraform-introduction.md** - What is IaC? Why Terraform?
2. **02-terraform-core-concepts.md** - Providers, Resources, State
3. **03-terraform-variables-outputs.md** - Dynamic configurations
4. **04-terraform-state-management.md** - Local vs Remote state, locking
5. **05-terraform-modules.md** - Creating reusable components
6. **06-terraform-workspaces.md** - Managing multiple environments
7. **07-terraform-best-practices.md** - Security, versioning, testing

### Part B: Ansible (Configuration Management)
8. **08-ansible-introduction.md** - Agentless automation philosophy
9. **09-ansible-inventory.md** - Defining target systems
10. **10-ansible-playbooks.md** - Writing automation scripts
11. **11-ansible-roles.md** - Modular playbook design
12. **12-ansible-templates-variables.md** - Dynamic configurations
13. **13-ansible-vault.md** - Secure credential management
14. **14-ansible-best-practices.md** - Idempotency, testing, debugging

### Hands-on Labs
- **lab-01**: Deploy VPC and EC2 instances with Terraform
- **lab-02**: Configure web servers with Ansible
- **lab-03**: Complete multi-tier application deployment
- **lab-04**: Implement infrastructure CI/CD pipeline

### Capstone Project
**Project**: Deploy a production-ready microservices infrastructure
- VPC with public/private subnets
- Auto-scaling group behind Application Load Balancer
- RDS database in private subnet
- Bastion host for secure access
- Automated configuration of all servers
- Monitoring and logging setup

## 🛠️ Prerequisites
- Basic Linux command line skills (Module 02)
- Understanding of cloud concepts (Module 08 helpful but not required)
- Git fundamentals (Module 03)
- AWS Free Tier account (or Azure/GCP equivalent)

## ⏱️ Estimated Time
- **Terraform Section**: 15-20 hours
- **Ansible Section**: 12-15 hours
- **Labs & Projects**: 10-15 hours
- **Total**: 37-50 hours (2-3 weeks)

## 📖 Additional Resources
- [Terraform Official Documentation](https://www.terraform.io/docs)
- [Ansible Official Documentation](https://docs.ansible.com)
- [Terraform Registry](https://registry.terraform.io)
- [Ansible Galaxy](https://galaxy.ansible.com)

## 🎓 Certification Path
- HashiCorp Certified: Terraform Associate
- Red Hat Certified Specialist in Ansible Automation

---

**Ready to start?** Begin with [01-terraform-introduction.md](terraform/01-terraform-introduction.md)
