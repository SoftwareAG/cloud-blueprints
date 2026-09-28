# Adabas & Natural on GCP – Advanced (GKE)

## Overview
The Advanced blueprint runs Adabas & Natural as containers on Google Kubernetes
Engine (GKE). It is intended for scalable, resilient, cloud-native workloads.
Customers provision the GKE cluster and supporting resources shown in the diagram
and deploy the official Software AG Adabas and Natural container images.

The reference layout places a regional GKE cluster in one GCP project, with worker
nodes spread across two zones of a single region (the diagram uses `europe-west3`,
zones `europe-west3-a` and `europe-west3-b`). Users reach the application over HTTPS
through Cloud Load Balancing. The customer's on-premises network connects privately
through Cloud Interconnect.

## Architecture Layers
- **Application layer (Natural):** Natural with Natural Web I/O (NWO) and NHA runs
  as a stateless Kubernetes Deployment. Replicas are spread across both zones, and
  the Cloud Load Balancer distributes HTTPS traffic between them. Scale it
  horizontally by adding replicas, or automatically with the Horizontal Pod
  Autoscaler.
- **Batch layer (Natural Batch):** Natural batch workloads run in the same pod as
  the database and use the local Adabas instance.
- **Data layer (Adabas):** Adabas (AEL) and the Adabas REST Server run as a
  stateful Kubernetes StatefulSet. It has a stable network identity and a
  Persistent Disk-backed PersistentVolume, so database files survive pod restarts
  and rescheduling.
- **Management layer (Adabas Manager):** a separate pod for web-based
  administration and monitoring of the Adabas databases.

## Zone Topology
| Zone             | Workloads                                                                                                |
| ---------------- | -------------------------------------------------------------------------------------------------------- |
| `europe-west3-a` | Natural + NWO / NHA; Adabas Manager; Natural Batch + Adabas (AEL) + Adabas REST Server; Persistent Disk  |
| `europe-west3-b` | Natural + NWO / NHA                                                                                      |

The Natural layer is active in both zones, so it keeps running if one zone fails.
The Adabas StatefulSet is pinned to one zone because its zonal Persistent Disk
lives there. To protect the data layer against a zone outage, use regional
Persistent Disks (synchronous replication across two zones) together with Backup
for GKE / Cloud Storage backups.

## Cloud & Security Resources
### Core platform

- **Google Kubernetes Engine (GKE)** – managed Kubernetes control plane with node
  pools across zones.
- **Cloud Load Balancing** – HTTPS entry point (via GKE Ingress/Gateway) with SSL
  termination and health-checked routing to Natural pods.
- **Persistent Disk** – durable block storage for Adabas data, and for Natural
  files where needed, provisioned through the Compute Engine PD CSI driver.
- **VPC & Subnets** – network isolation for the cluster, nodes, pods and services.

### Hybrid connectivity

- **Cloud Interconnect** – private, high-bandwidth link from the customer network
  (on-premises router) to GCP.
- **Cloud Router** – dynamic BGP routing between the customer network and the VPC.

### Supporting services

- **Memorystore** – managed in-memory cache for session or application data.
- **Artifact Registry / Container Registry** – private registry for the Adabas and
  Natural container images.
- **Cloud Key Management Service (KMS)** – customer-managed encryption keys for
  Persistent Disks, Kubernetes Secrets and backups.
- **Cloud Logging & Cloud Monitoring** – centralized logs, metrics, dashboards and
  alerting for the cluster and workloads.
- **Backup for GKE** – backup and restore of cluster resources and persistent
  volumes.
- **Cloud Storage** – object storage for database backups, archives and exports.

### Security controls

- **Network Policies** – restrict pod-to-pod traffic so only the Natural layer and
  Adabas Manager can reach the Adabas layer.
- **Workload Identity** – least-privilege pod access to GCP services (backups,
  monitoring, KMS) without service-account keys.
- **OIDC** – single sign-on for cluster access and application users through the
  organization's identity provider.
- **Secrets & ConfigMaps** – manage credentials and configuration securely
  (optionally synced from Secret Manager).

## Deployment & Operations Tooling
- **Terraform** – provisions the GCP project resources as infrastructure as code:
  VPC, GKE cluster, node pools, load balancer, KMS, storage and connectivity.
- **GitOps** – declarative delivery of the Kubernetes manifests (Deployments,
  StatefulSet, Services, Ingress, policies) from Git, for example with Config Sync
  or Argo CD.
- **OIDC** – federated authentication for operators and CI/CD pipelines.

## Traffic Flow
1. Users connect over **HTTPS** to the **Cloud Load Balancer**.
2. The load balancer forwards requests to healthy **Natural + NWO / NHA** pods in
   either zone.
3. Natural pods reach **Adabas** over the cluster network, restricted by Network
   Policies.
4. Adabas persists data to **Persistent Disk**. Backups go to **Cloud Storage** /
   **Backup for GKE**, encrypted with **Cloud KMS** keys.
5. On-premises systems reach the cluster privately through **Cloud Interconnect**
   and **Cloud Router**.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on GCP - GKE.gif`

## Intended Use
Production, QA, and advanced development environments needing scalability,
resilience, and cloud-native operations. For non-containerized deployments use the
Basic or Resilient HA blueprints.
