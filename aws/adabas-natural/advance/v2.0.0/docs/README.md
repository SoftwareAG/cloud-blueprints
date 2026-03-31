## References
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Software AG Container Images](https://hub.docker.com/u/softwareag)
- [an-k8-local GitHub Repository](https://github.com/SoftwareAG/an-k8-local)
# Adabas and Natural on AWS EKS – Advanced Containerized Deployment

## Overview
This architecture describes an advanced deployment of Adabas and Natural on AWS using the Elastic Kubernetes Service (EKS) and official Software AG container images. It is designed for scalable, resilient, and cloud-native enterprise workloads, leveraging Kubernetes orchestration, automated scaling, and robust security.


## Architecture Components

- **AWS EKS Cluster**: Managed Kubernetes service hosting Adabas and Natural containers across multiple worker nodes and Availability Zones.
- **Software AG Container Images**: Official images for Adabas and Natural, pulled from a private registry (e.g., Azure Container Registry or AWS ECR).
- **Natural Deployment (Stateless)**: Natural is deployed as a Kubernetes Deployment, providing stateless, scalable application pods.
- **Adabas StatefulSet (Stateful)**: Adabas is deployed as a Kubernetes StatefulSet, ensuring persistent identity, storage, and reliable database operations.
- **Persistent Volumes (EBS-backed)**: Provide durable storage for Adabas databases and Natural application data.
- **Kubernetes Services**: Expose Natural and Adabas endpoints internally and externally, with support for load balancing.
- **Ingress Controller**: Manages external access, SSL termination, and routing for Natural web applications.
- **Network Policies**: Enforce traffic rules between pods, namespaces, and external resources for security and compliance.
- **IAM Roles for Service Accounts (IRSA)**: Securely grant pods access to AWS resources (e.g., S3 for backup, CloudWatch for monitoring).
- **Secrets & ConfigMaps**: Manage sensitive data and configuration for containers.

## Key Features
- **Multi-AZ High Availability**: EKS nodes and persistent volumes span multiple Availability Zones for fault tolerance.
- **Automated Scaling**: Horizontal Pod Autoscaler and Cluster Autoscaler adjust resources based on demand.
- **Rolling Updates & Self-Healing**: Kubernetes automatically replaces failed pods and supports zero-downtime upgrades.
- **Secure Image Pulls**: Use imagePullSecrets for private registry authentication.
- **Centralized Monitoring & Logging**: Integrate with AWS CloudWatch, Prometheus, and Grafana for observability.

## Deployment Steps

1. **Provision EKS Cluster**
   - Use Terraform or AWS CLI to create an EKS cluster with worker nodes across multiple AZs.
   - Configure networking (VPC, subnets, security groups).

2. **Configure Storage**
   - Set up EBS-backed StorageClasses and PersistentVolumeClaims for Adabas and Natural data.

3. **Deploy Container Images**
   - Create Kubernetes Deployments/StatefulSets for Adabas and Natural using official Software AG images.
   - Reference imagePullSecrets for private registry access.

4. **Expose Services**
   - Create Kubernetes Services (ClusterIP, NodePort, or LoadBalancer) for Adabas and Natural endpoints.
   - Deploy an Ingress Controller for external access and SSL termination.

5. **Configure Security**
   - Apply NetworkPolicies to restrict pod communication.
   - Use IRSA for secure AWS resource access.
   - Store credentials and config in Secrets and ConfigMaps.

6. **Enable Monitoring & Logging**
   - Deploy Prometheus and Grafana for metrics.
   - Integrate with AWS CloudWatch for logs and alerts.

7. **Automate with Terraform**
   - Use provided Terraform scripts to automate cluster, storage, and workload provisioning.

## Advanced Features
- **Backup & Restore**: Automate database backups to S3 using Kubernetes CronJobs and IRSA.
- **Disaster Recovery**: Multi-AZ deployment and persistent storage ensure rapid recovery from failures.
- **CI/CD Integration**: Use GitOps or CI/CD pipelines for automated image builds and deployments.
- **Custom Scaling Policies**: Tune autoscalers for optimal performance and cost.

## Diagram Reference
The architecture diagram in this folder illustrates:
- EKS cluster spanning multiple AZs
- Adabas and Natural pods running on worker nodes
- Persistent storage with EBS
- Load balancer and ingress for external access
- Network segmentation and security controls

## Intended Use
This setup is recommended for production, QA, and advanced development environments requiring scalability, resilience, and cloud-native features. For simpler or non-containerized deployments, see the basic and resilient HA architectures in this repository.

---

**Note:** All Kubernetes manifest files for this deployment are available in the `manifests` folder at this level. Use these files to deploy Adabas and Natural workloads, services, ingress, and other resources to your EKS cluster.
