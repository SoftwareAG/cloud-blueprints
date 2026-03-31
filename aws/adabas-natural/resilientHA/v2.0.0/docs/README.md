# Adabas and Natural on AWS Resilient HA Setup – Multi-Instance, Load Balanced

## Overview
This architecture provides a resilient, highly available setup for deploying Adabas and Natural on AWS, using multiple EC2 instances distributed across Availability Zones and fronted by a load balancer. It is designed for production workloads, business continuity, and fault tolerance, while maintaining security and operational best practices.



## Architecture Components

- **AWS EC2 Instances (Multi-AZ)**: Multiple EC2 instances for Adabas and Natural are deployed across different AWS Availability Zones to ensure high availability and fault tolerance.
- **Elastic Load Balancer (ELB)**: Distributes incoming application traffic across Natural EC2 instances, providing seamless failover and scaling.
- **Amazon EBS Volumes**: Attached to each EC2 instance for persistent storage of Adabas databases and Natural application files. For Adabas, consider using EBS Multi-Attach or replication for HA.
- **Security Groups**: Control inbound and outbound traffic to EC2 instances and the load balancer. Only necessary ports (e.g., SSH, Adabas, Natural, HTTP/S) are opened.
- **IAM Roles**: Attached to EC2 instances for secure access to AWS resources (e.g., S3 for backup/restore, CloudWatch for monitoring).
- **VPC/Subnets (Multi-AZ)**: The architecture uses multiple subnets in different Availability Zones for network isolation and redundancy.


---

## Resilient HA Architecture: Multi-Instance, Load Balanced, Multi-AZ

This setup extends the distributed architecture by introducing high availability and load balancing:

- **Adabas EC2 Instances (Multi-AZ):**
  Adabas database nodes are deployed in multiple Availability Zones. Data replication or clustering is recommended for database HA.

- **Natural EC2 Instances (Multi-AZ):**
  Natural application nodes are deployed in multiple Availability Zones and registered with the load balancer.

- **Elastic Load Balancer (ELB):**
  The ELB routes client requests to healthy Natural nodes, automatically failing over if a node becomes unavailable.

- **Network Segregation:**
  Separate subnets and security groups for database and application layers, with strict access controls.

- **Automated Failover:**
  If an EC2 instance or AZ fails, traffic is automatically redirected to healthy instances in other AZs.

- **Scalability:**
  Easily add more Natural nodes behind the load balancer for horizontal scaling.

### Key Benefits
- **High Availability:** No single point of failure; resilient to instance or AZ outages.
- **Performance:** Load balancer distributes traffic for optimal resource usage.
- **Security:** Network and IAM controls for each layer.
- **Operational Efficiency:** Automated provisioning and scaling with Terraform.

### Deployment Steps (HA)
1. **Provision VPC and Subnets in Multiple AZs**
2. **Deploy Adabas EC2 nodes in separate AZs**
3. **Deploy Natural EC2 nodes in separate AZs**
4. **Create and configure Elastic Load Balancer**
5. **Attach EBS volumes to each node**
6. **Configure Security Groups for each layer and ELB**
7. **Set up IAM roles and monitoring**
8. **Automate with Terraform scripts**

### Use Cases
- Production workloads requiring high availability and fault tolerance
- Business continuity and disaster recovery
- Scalable, secure enterprise deployments

---
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
