# Adabas and Natural on GCP Basic Setup – Development Single Instance

## Overview
This architecture provides a basic setup for deploying Adabas and Natural on Google Cloud Platform (GCP) for development purposes, using a single Compute Engine instance. It is designed for simplicity, rapid prototyping, and easy access for developers, while maintaining essential security and operational best practices.


## Architecture Components

- **GCP Compute Engine Instance**: Hosts both Adabas and Natural environments. The machine type should be selected based on development workload requirements (e.g., e2-standard-2 or larger).
- **Persistent Disks**: Attached to the Compute Engine instance for persistent storage of Adabas databases and Natural application files.
- **Firewall Rules**: Control inbound and outbound traffic to the Compute Engine instance. Only necessary ports (e.g., SSH, Adabas, Natural) are opened for development access.
- **IAM Service Account**: Optionally attached to the Compute Engine instance for secure access to GCP resources (e.g., Cloud Storage for backup/restore).
- **External IP (optional)**: For static public access to the instance if required.
- **VPC/Subnet**: The instance is deployed in a dedicated VPC and subnet for network isolation.

---

## Distributed Architecture: Adabas and Natural on Separate Compute Engine Instances

In addition to the single-instance setup, this repository also supports a distributed architecture where Adabas (database layer) and Natural (application layer) are hosted on separate Compute Engine instances. This design provides:

- **Data and Application Layer Segregation:**
   Adabas runs on its own Compute Engine instance, while Natural runs on a separate instance. This separation improves security, scalability, and maintainability.

- **Network Segregation:**
   Each layer can be placed in its own subnet or firewall scope, allowing fine-grained control over network access. For example, only the Natural instance can access Adabas, and only trusted sources can access Natural.

- **Scalability:**
   You can independently scale the database and application layers based on workload requirements.

- **High Availability and Fault Isolation:**
   Issues in the application layer do not directly impact the database layer and vice versa.

### Key Components

- **Adabas Compute Engine Instance:**
   Dedicated to running the Adabas database, with attached Persistent Disks for persistent storage.

- **Natural Compute Engine Instance:**
   Hosts the Natural runtime and applications, with its own Persistent Disks.

- **Firewall Rules and Subnets:**
   Separate firewall rules and subnets for each layer, enforcing network policies and isolation.

- **IAM Service Accounts:**
   Each instance can have its own service account for secure access to GCP resources.

### Deployment

- Use the provided Terraform scripts to define multiple Compute Engine instances, firewall rules, and subnets.
- Configure firewall rules to allow only necessary traffic between the Natural and Adabas instances.
- Attach and format Persistent Disks for each instance as required.

### Use Cases

- Recommended for environments where data security, compliance, or scalability are priorities.
- Suitable for production, QA, or advanced development setups.

---

## Deployment Steps

1. **Launch Compute Engine Instance**
   - Choose a supported public image (e.g., RHEL or SUSE).
   - Select machine type (minimum e2-standard-2 recommended).
   - Attach IAM service account if needed.
   - Reserve a static external IP if public access is required.

2. **Attach Persistent Disks**
   - Create and attach Persistent Disks for Adabas data and Natural application files.
   - Format and mount disks on the instance.

3. **Configure Firewall Rules**
   - Allow SSH (port 22) from trusted IPs.
   - Open Adabas and Natural ports as needed (e.g., 60001, 2700, 8080).
   - Restrict all other traffic.

4. **Install Adabas and Natural**
   - SSH into the instance.
   - Install Adabas and Natural using provided installation scripts or manual steps.
   - Configure Adabas database and Natural runtime environment.

5. **Backup and Restore (Optional)**
   - Use Cloud Storage for storing backups.
   - Configure scripts to automate backup/restore operations.

## Security Considerations
- Restrict SSH access to trusted IPs only.
- Use IAM service accounts for secure GCP resource access.
- Regularly update the instance and software for security patches.
- Enable Customer-Managed Encryption Keys (CMEK) on Persistent Disks for data protection.

## Maintenance and Operations
- Monitor instance health and resource usage with Cloud Monitoring.
- Schedule regular backups of Adabas and Natural data.
- Use OS Config and VM Manager for patching and automation.

## Diagram Reference
This setup is visually represented in the architecture diagrams provided in the `docs` folder:

- **`Adabas and Natural on GCP Basic Setup Development Single Instance-v2.0.0.gif`** – single Compute Engine instance hosting both Adabas and Natural, showing:
   - Compute Engine instance hosting Adabas and Natural
   - Persistent Disks attached for persistent storage
   - Firewall rule boundaries
   - Optional external IP for public access
   - VPC/subnet for network isolation

- **`Adabas and Natural on GCP Basic Setup Distributed-v2.0.0.gif`** – distributed setup with Adabas and Natural on separate Compute Engine instances, showing:
   - Dedicated Adabas and Natural Compute Engine instances
   - Persistent Disks attached to each instance
   - Separate firewall rules and subnets per layer
   - Controlled network access between the Natural and Adabas instances
   - VPC/subnets for network isolation

## Intended Use
This architecture is intended for development and prototyping. For production or high availability, consider the advanced or resilient HA architectures provided in this repository.

---
## Automating Provisioning with Terraform

You can automate the entire infrastructure provisioning for this architecture using Terraform scripts provided in the `parameters` and `templates` folders. Terraform enables you to define infrastructure as code, making deployments repeatable, version-controlled, and easy to update.

### Key Concepts
- **Terraform Modules**: Reusable building blocks for Compute Engine, Persistent Disks, Firewall Rules, IAM service accounts, and networking.
- **Variables**: Parameterize your deployment (machine type, disk size, VPC ID, etc.) for flexibility.
- **State Management**: Terraform tracks resources in a state file, allowing safe updates and destruction.
- **Outputs**: Automatically display important information (instance IP, Persistent Disk IDs, etc.) after deployment.

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
   - Terraform will display instance details, external IP, and other outputs for easy access.

### What Gets Automated
- Compute Engine instance creation and configuration
- Persistent Disk creation, attachment, and mounting
- Firewall rule setup
- IAM service account assignment
- VPC and subnet configuration
- (Optional) Cloud Storage bucket for backups

### Customization
You can extend the Terraform scripts to:
- Add more instances or environments
- Integrate with CI/CD pipelines
- Automate software installation using startup scripts or provisioners
- Schedule backups and monitoring

---
For more details, see the installation scripts and parameter files in the `scripts` and `parameters` folders.
This architecture is intended for development and prototyping. For production or high availability, consider the advanced or resilient HA architectures provided in this repository.

---
For more details, see the installation scripts and parameter files in the `scripts` and `parameters` folders.
