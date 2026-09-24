# Adabas and Natural on GCP Resilient HA Setup – Multi-Instance, Load Balanced

## Overview
This architecture provides a resilient, highly available setup for deploying Adabas and Natural on Google Cloud Platform (GCP), using multiple Compute Engine instances distributed across zones and fronted by a load balancer. It is designed for production workloads, business continuity, and fault tolerance, while maintaining security and operational best practices.



## Architecture Components

- **GCP Compute Engine Instances (Multi-Zone)**: Multiple Compute Engine instances for Adabas and Natural are deployed across different GCP zones within a region to ensure high availability and fault tolerance.
- **Cloud Load Balancing**: Distributes incoming application traffic across Natural instances, providing seamless failover and scaling.
- **Persistent Disks**: Attached to each Compute Engine instance for persistent storage of Adabas databases and Natural application files. For Adabas, consider using regional Persistent Disks or replication for HA.
- **Firewall Rules**: Control inbound and outbound traffic to Compute Engine instances and the load balancer. Only necessary ports (e.g., SSH, Adabas, Natural, HTTP/S) are opened.
- **IAM Service Accounts**: Attached to Compute Engine instances for secure access to GCP resources (e.g., Cloud Storage for backup/restore, Cloud Monitoring for observability).
- **VPC/Subnets (Multi-Zone)**: The architecture uses a regional subnet spanning multiple zones for network isolation and redundancy.


---

## Resilient HA Architecture: Multi-Instance, Load Balanced, Multi-Zone

This setup extends the distributed architecture by introducing high availability and load balancing:

- **Adabas Compute Engine Instances (Multi-Zone):**
  Adabas database nodes are deployed in multiple zones. Data replication or clustering is recommended for database HA.

- **Natural Compute Engine Instances (Multi-Zone):**
  Natural application nodes are deployed in multiple zones and registered with the load balancer backend service.

- **Cloud Load Balancing:**
  The load balancer routes client requests to healthy Natural nodes, automatically failing over if a node becomes unavailable.

- **Network Segregation:**
  Separate subnets and firewall rules for database and application layers, with strict access controls.

- **Automated Failover:**
  If a Compute Engine instance or zone fails, traffic is automatically redirected to healthy instances in other zones.

- **Scalability:**
  Easily add more Natural nodes to a managed instance group behind the load balancer for horizontal scaling.

### Key Benefits
- **High Availability:** No single point of failure; resilient to instance or zone outages.
- **Performance:** Load balancer distributes traffic for optimal resource usage.
- **Security:** Network and IAM controls for each layer.
- **Operational Efficiency:** Automated provisioning and scaling with the `gcloud` deployment scripts.

### Deployment Steps (HA)
1. **Provision VPC and Subnets across Multiple Zones**
2. **Deploy Adabas Compute Engine nodes in separate zones**
3. **Deploy Natural Compute Engine nodes in separate zones**
4. **Create and configure Cloud Load Balancing**
5. **Attach Persistent Disks to each node**
6. **Configure Firewall Rules for each layer and the load balancer**
7. **Set up IAM service accounts and monitoring**
8. **Automate with the provided `gcloud` scripts**

### Use Cases
- Production workloads requiring high availability and fault tolerance
- Business continuity and disaster recovery
- Scalable, secure enterprise deployments

---
---

## Deployment Steps

1. **Launch Compute Engine Instances**
   - Choose a supported public image (e.g., RHEL or SUSE).
   - Select machine type (minimum e2-standard-2 recommended).
   - Attach IAM service account if needed.
   - Distribute instances across multiple zones.

2. **Attach Persistent Disks**
   - Create and attach Persistent Disks for Adabas data and Natural application files.
   - Format and mount disks on each instance.

3. **Configure Firewall Rules**
   - Allow SSH (port 22) from trusted IPs.
   - Open Adabas and Natural ports as needed (e.g., 60001, 2700, 8080).
   - Restrict all other traffic.

4. **Install Adabas and Natural**
   - SSH into the instances.
   - Install Adabas and Natural using provided installation scripts or manual steps.
   - Configure Adabas database and Natural runtime environment.

5. **Backup and Restore (Optional)**
   - Use Cloud Storage for storing backups.
   - Configure scripts to automate backup/restore operations.

## Security Considerations
- Restrict SSH access to trusted IPs only.
- Use IAM service accounts for secure GCP resource access.
- Regularly update the instances and software for security patches.
- Enable Customer-Managed Encryption Keys (CMEK) on Persistent Disks for data protection.

## Maintenance and Operations
- Monitor instance health and resource usage with Cloud Monitoring.
- Schedule regular backups of Adabas and Natural data.
- Use OS Config and VM Manager for patching and automation.

## Diagram Reference
This setup is visually represented in the provided architecture diagram (`Adabas and Natural on GCP Resilient HA - Detailed Security-v2.0.0.gif`) in the `docs` folder. The diagram shows:
- Compute Engine instances hosting Adabas and Natural across multiple zones
- Persistent Disks attached for persistent storage
- Cloud Load Balancing distributing application traffic
- Firewall rule boundaries and network segmentation between layers
- IAM service accounts and encryption controls for secure access
- VPC/subnets for network isolation and redundancy

## Intended Use
This architecture is intended for production workloads requiring high availability. For simpler development setups, see the basic architecture, or for cloud-native container deployments, see the advanced architecture provided in this repository.

---
## Automating Provisioning

This architecture can be provisioned with `gcloud` CLI deployment scripts placed in the `scripts` folder. The scripts are intended to create the network, firewall rules, service account, the autoscaled Natural managed instance group with its load balancer, and the per-zone Adabas nodes in a repeatable, parameterized way.

### Suggested Workflow
1. **Configure**
   - Set your deployment parameters (`PROJECT_ID`, trusted SSH CIDRs — never use `0.0.0.0/0`, machine types, replica counts, disk sizes) and provide at least two zones for high availability.

2. **Authenticate**
   - Run `gcloud auth login` and ensure the target project has billing and the Compute Engine API enabled.

3. **Deploy**
   - Run the deployment script to create the VPC, subnet, Cloud NAT, firewall rules, service account, Natural instance template + regional MIG + autoscaler + external load balancer, and the Adabas nodes with data disks.

4. **Tear Down**
   - Run the cleanup script to delete the deployment.

### What Should Be Automated
- Compute Engine instance creation and configuration across zones
- Autoscaled Natural managed instance group and external load balancer
- Persistent Disk creation, attachment, and mounting (via a VM startup script)
- Firewall rule setup (least privilege, including Google health-check ranges)
- IAM service account assignment
- VPC, subnet, and Cloud NAT configuration
