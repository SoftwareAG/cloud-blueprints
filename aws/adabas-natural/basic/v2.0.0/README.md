# Adabas and Natural on AWS Basic Setup – Development Single Instance

## Overview
This architecture provides a basic setup for deploying Adabas and Natural on AWS for development purposes, using a single EC2 instance. It is designed for simplicity, rapid prototyping, and easy access for developers, while maintaining essential security and operational best practices.


## Architecture Components

- **AWS EC2 Instance**: Hosts both Adabas and Natural environments. The instance type should be selected based on development workload requirements (e.g., t3.medium or larger).
- **Amazon EBS Volumes**: Attached to the EC2 instance for persistent storage of Adabas databases and Natural application files.
- **Security Group**: Controls inbound and outbound traffic to the EC2 instance. Only necessary ports (e.g., SSH, Adabas, Natural) are opened for development access.
- **IAM Role**: Optionally attached to the EC2 instance for secure access to AWS resources (e.g., S3 for backup/restore).
- **Elastic IP (optional)**: For static public access to the instance if required.
- **VPC/Subnet**: The instance is deployed in a dedicated VPC and subnet for network isolation.

---

## Distributed Architecture: Adabas and Natural on Separate EC2 Instances

In addition to the single-instance setup, this repository also supports a distributed architecture where Adabas (database layer) and Natural (application layer) are hosted on separate EC2 instances. This design provides:

- **Data and Application Layer Segregation:**
   Adabas runs on its own EC2 instance, while Natural runs on a separate instance. This separation improves security, scalability, and maintainability.

- **Network Segregation:**
   Each layer can be placed in its own subnet or security group, allowing fine-grained control over network access. For example, only the Natural instance can access Adabas, and only trusted sources can access Natural.

- **Scalability:**
   You can independently scale the database and application layers based on workload requirements.

- **High Availability and Fault Isolation:**
   Issues in the application layer do not directly impact the database layer and vice versa.

### Key Components

- **Adabas EC2 Instance:**
   Dedicated to running the Adabas database, with attached EBS volumes for persistent storage.

- **Natural EC2 Instance:**
   Hosts the Natural runtime and applications, with its own EBS volumes.

- **Security Groups and Subnets:**
   Separate security groups and subnets for each layer, enforcing network policies and isolation.

- **IAM Roles:**
   Each instance can have its own IAM role for secure access to AWS resources.

### Deployment

- Use the provided Terraform scripts to define multiple EC2 instances, security groups, and subnets.
- Configure security group rules to allow only necessary traffic between the Natural and Adabas instances.
- Mount and format EBS volumes for each instance as required.

### Use Cases

- Recommended for environments where data security, compliance, or scalability are priorities.
- Suitable for production, QA, or advanced development setups.

---

## Deployment Steps

1. **Launch EC2 Instance**
   - Choose an Amazon Linux 2 or RHEL AMI.
   - Select instance type (minimum t3.medium recommended).
   - Attach IAM role if needed.
   - Assign Elastic IP if public access is required.

2. **Attach EBS Volumes**
   - Create and attach EBS volumes for Adabas data and Natural application files.
   - Format and mount volumes on the instance.

3. **Configure Security Group**
   - Allow SSH (port 22) from trusted IPs.
   - Open Adabas and Natural ports as needed (e.g., 60001, 2700, 8080).
   - Restrict all other traffic.

4. **Install Adabas and Natural**
   - SSH into the instance.
   - Install Adabas and Natural using provided installation scripts or manual steps.
   - Configure Adabas database and Natural runtime environment.

5. **Backup and Restore (Optional)**
   - Use AWS S3 for storing backups.
   - Configure scripts to automate backup/restore operations.

## Security Considerations
- Restrict SSH access to trusted IPs only.
- Use IAM roles for secure AWS resource access.
- Regularly update the instance and software for security patches.
- Encrypt EBS volumes for data protection.

## Maintenance and Operations
- Monitor instance health and resource usage with AWS CloudWatch.
- Schedule regular backups of Adabas and Natural data.
- Use AWS Systems Manager for patching and automation.

## Diagram Reference
This setup is visually represented in the provided architecture diagram (`Adabas and Natural on AWS Basic Setup Development Single Instance_v1.0.0.gif`). The diagram shows:
- EC2 instance hosting Adabas and Natural
- EBS volumes attached for persistent storage
- Security group boundaries
- Optional Elastic IP for public access
- VPC/subnet for network isolation

## Intended Use
This architecture is intended for development and prototyping. For production or high availability, consider the advanced or resilient HA architectures provided in this repository.

---
## Automating Provisioning with Terraform

You can automate the entire infrastructure provisioning for this architecture using Terraform scripts provided in the `parameters` and `templates` folders. Terraform enables you to define infrastructure as code, making deployments repeatable, version-controlled, and easy to update.

### Key Concepts
- **Terraform Modules**: Reusable building blocks for EC2, EBS, Security Groups, IAM roles, and networking.
- **Variables**: Parameterize your deployment (instance type, volume size, VPC ID, etc.) for flexibility.
- **State Management**: Terraform tracks resources in a state file, allowing safe updates and destruction.
- **Outputs**: Automatically display important information (instance IP, EBS volume IDs, etc.) after deployment.

### Example Workflow
1. **Configure Variables**
   - Edit the variable files in the `parameters` folder to match your environment and requirements.

2. **Initialize Terraform**
   - Run `terraform init` in the `templates` directory to download required providers and modules.

3. **Plan Deployment**
   - Run `terraform plan -var-file=../parameters/dev.tfvars` to preview changes.

4. **Apply Deployment**
   - Run `terraform apply -var-file=../parameters/dev.tfvars` to create all resources automatically.

5. **Access Outputs**
   - Terraform will display instance details, public IP, and other outputs for easy access.

### What Gets Automated
- EC2 instance creation and configuration
- EBS volume creation, attachment, and mounting
- Security group setup
- IAM role assignment
- VPC and subnet configuration
- (Optional) S3 bucket for backups

### Customization
You can extend the Terraform scripts to:
- Add more instances or environments
- Integrate with CI/CD pipelines
- Automate software installation using user data or provisioners
- Schedule backups and monitoring

---
For more details, see the installation scripts and parameter files in the `scripts` and `parameters` folders.
This architecture is intended for development and prototyping. For production or high availability, consider the advanced or resilient HA architectures provided in this repository.

---
For more details, see the installation scripts and parameter files in the `scripts` and `parameters` folders.
