# Adabas & Natural on AWS – Advanced (EKS)

## Overview
The Advanced blueprint runs Adabas & Natural as containers on Amazon EKS
(Kubernetes). It is intended for scalable, resilient, cloud-native workloads.
Customers provision the EKS cluster and supporting resources shown in the diagrams
and deploy the official Software AG Adabas and Natural container images.

## Architecture Layers
- **Application layer (Natural):** deployed as a stateless Kubernetes Deployment,
  allowing horizontal scaling of application pods across worker nodes and AZs.
- **Data layer (Adabas):** deployed as a stateful Kubernetes StatefulSet with
  persistent identity and EBS-backed persistent volumes for reliable database
  operation.

## Cloud & Security Resources
- **Amazon EKS** – managed Kubernetes control plane and worker nodes across AZs.
- **Persistent Volumes (EBS)** – durable storage for Adabas data and Natural files.
- **Kubernetes Services & Ingress** – expose the layers internally/externally with
  load balancing and SSL termination.
- **Network Policies** – restrict pod-to-pod traffic so only the Natural layer can
  reach the Adabas layer.
- **VPC & Subnets** – network isolation for the cluster and node groups.
- **IAM Roles for Service Accounts (IRSA)** – least-privilege pod access to AWS
  services (backups, monitoring).
- **Secrets & ConfigMaps** – manage credentials and configuration securely.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on  AWS Advance - EKS part1_v2.0.0.gif`
- `Adabas and Natural on  AWS Advance - EKS part2_v2.0.0.gif`

## Intended Use
Production, QA, and advanced development environments needing scalability,
resilience, and cloud-native operations. For non-containerized deployments use the
Basic or Resilient HA blueprints.
