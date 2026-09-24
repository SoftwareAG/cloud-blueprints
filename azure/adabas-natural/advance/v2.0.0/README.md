# Adabas & Natural on Azure – Advanced (AKS)

## Overview
The Advanced blueprint runs Adabas & Natural as containers on Azure Kubernetes
Service (AKS). It is intended for scalable, resilient, cloud-native workloads.
Customers provision the AKS cluster and supporting resources shown in the diagram
and deploy the official Software AG Adabas and Natural container images.

## Architecture Layers
- **Application layer (Natural):** deployed as a stateless Kubernetes Deployment,
  allowing horizontal scaling of application pods across worker nodes and zones.
- **Data layer (Adabas):** deployed as a stateful Kubernetes StatefulSet with
  persistent identity and Managed Disk-backed persistent volumes for reliable
  database operation.

## Cloud & Security Resources
- **Azure Kubernetes Service (AKS)** – managed Kubernetes control plane and node
  pools across Availability Zones.
- **Persistent Volumes (Managed Disks)** – durable storage for Adabas data and
  Natural files.
- **Kubernetes Services & Ingress** – expose the layers internally/externally with
  load balancing and SSL termination.
- **Network Policies** – restrict pod-to-pod traffic so only the Natural layer can
  reach the Adabas layer.
- **Virtual Network (VNet) & Subnets** – network isolation for the cluster and node
  pools.
- **Managed Identity** – least-privilege pod access to Azure services (backups,
  monitoring).
- **Secrets & ConfigMaps** – manage credentials and configuration securely.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on Azure - AKS-v2.0.0.gif`

## Intended Use
Production, QA, and advanced development environments needing scalability,
resilience, and cloud-native operations. For non-containerized deployments use the
Basic or Resilient HA blueprints.
