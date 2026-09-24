# Adabas & Natural on GCP – Advanced (GKE)

## Overview
The Advanced blueprint runs Adabas & Natural as containers on Google Kubernetes
Engine (GKE). It is intended for scalable, resilient, cloud-native workloads.
Customers provision the GKE cluster and supporting resources and deploy the official
Software AG Adabas and Natural container images.

## Architecture Layers
- **Application layer (Natural):** deployed as a stateless Kubernetes Deployment,
  allowing horizontal scaling of application pods across worker nodes and zones.
- **Data layer (Adabas):** deployed as a stateful Kubernetes StatefulSet with
  persistent identity and Persistent Disk-backed volumes for reliable database
  operation.

## Cloud & Security Resources
- **Google Kubernetes Engine (GKE)** – managed Kubernetes control plane and node
  pools across zones.
- **Persistent Volumes (Persistent Disks)** – durable storage for Adabas data and
  Natural files.
- **Kubernetes Services & Ingress** – expose the layers internally/externally with
  load balancing and SSL termination.
- **Network Policies** – restrict pod-to-pod traffic so only the Natural layer can
  reach the Adabas layer.
- **VPC & Subnets** – network isolation for the cluster and node pools.
- **Workload Identity** – least-privilege pod access to GCP services (backups,
  monitoring).
- **Secrets & ConfigMaps** – manage credentials and configuration securely.

## Architecture Diagrams
Diagrams for this blueprint are stored in the `docs` folder.

## Intended Use
Production, QA, and advanced development environments needing scalability,
resilience, and cloud-native operations. For non-containerized deployments use the
Basic or Resilient HA blueprints.
