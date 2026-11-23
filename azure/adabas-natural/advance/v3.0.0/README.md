# Adabas & Natural on Azure – Advanced (AKS)

## Overview

The Advanced blueprint runs Adabas & Natural as containers on Azure Kubernetes
Service (AKS) across three Availability Zones in one region, in a hub-and-spoke
network. It is intended for scalable, resilient, cloud-native production workloads.
It follows the v3.0.0 Resilient HA reference architecture, with the virtual machines
replaced by Kubernetes workloads. Customers provision the AKS cluster and supporting
resources shown in the diagram and deploy the official Software AG Adabas and Natural
container images.

## What's New in v3.0.0

- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI, deployed as replicas in all three zones.
- **Natural Subprogram Server** – runs Natural subprograms on behalf of the Natural
  OpenAPI Server, deployed as its own Deployment behind its own Service.
- **Natural Availability Server** – replicas in all three zones, with Natural session
  affinity kept in Azure Cache for Redis.
- **Application Gateway as ingress** – Azure Application Gateway + WAF, managed by the
  Application Gateway Ingress Controller (AGIC), is the single public HTTPS entry
  point, with one endpoint for the Natural OpenAPI Server and one for the Natural
  Availability Server.
- **3-zone Adabas cluster** – a 3-pod Adabas cluster StatefulSet with one pod per
  Availability Zone and SYNC replication between all three pods.

## Reference Architecture

### Ingress (hub VNet)

Azure Application Gateway + WAF (v2, zone-redundant) receives public HTTPS traffic
from browser, mobile, and API clients; internal and admin traffic comes in privately
over ExpressRoute / VPN. Through AGIC the gateway publishes two endpoints, each routed
to its own Kubernetes Service:

| Gateway endpoint (example) | Kubernetes Service     | Backend pods                |
| -------------------------- | ---------------------- | --------------------------- |
| `https://api.example.com`  | `natural-openapi`      | Natural OpenAPI Server      |
| `https://nas.example.com`  | `natural-availability` | Natural Availability Server |

Each endpoint has its own listener, TLS certificate (from Key Vault), WAF policy, and
health probe, so API traffic and online user traffic can be secured, scaled, and
monitored independently. Replace the example host names with the customer's own DNS
names; path-based routing on a single host name (for example `/api` and `/nas`) works
the same way.

AGIC sends traffic straight to pod IPs, so the cluster must use **Azure CNI**
networking for the gateway in the hub to reach the pods across the peering.
**Application Gateway for Containers** is Microsoft's newer ingress option for AKS
and can replace AGIC in new deployments.

### Web tier (`adanat-web` namespace, zones 1–3)

- **Natural OpenAPI Server** and **Natural Availability Server (NAS)** Deployments,
  one replica per zone, each behind its own Service.
- NAS keeps Natural session affinity in Azure Cache for Redis (zone-redundant), so a
  user's session survives the loss of a NAS pod or zone.

### App tier (`adanat` namespace, zones 1–3)

Two internal Services (ClusterIP) connect the web tier to the app tier:

- **`natural-subprogram`** → **Natural Subprogram Server** pods (Natural Subprogram
  Server, Natural Security, Natural), serving calls from the Natural OpenAPI Server.
- **`natural-webio`** → **Natural Web I/O Server** pods (Natural Web I/O Server,
  Natural Security, Natural), serving the online applications behind NAS.

Both run one replica per zone, so every tier keeps a healthy target if a zone is lost.

### Data tier (`adanat` namespace, zones 1–3)

- **Adabas Cluster** – a 3-replica **StatefulSet** (`adabas-0`, `adabas-1`,
  `adabas-2`), one pod per zone. Each pod runs Adabas with AEL and the Adabas Rest
  Server, and has its own zonal Premium SSD persistent volume (shared-nothing).
- **Primary and secondaries:** one pod is the primary and takes all read/write (R/W)
  calls over **ADATCP/S**; the other two are secondaries that also serve reads.
  **SYNC replication** runs between all three pods. The diagram shows `adabas-0` as
  the current primary, but the role moves on failover; Natural's Adabas client
  locates the primary, so Natural reconnects without configuration changes.
- **Natural Batch** – a single-replica Deployment (or a CronJob for scheduled jobs)
  that connects to the primary through the Adabas client. It is not pinned to a zone;
  if its pod, node, or zone fails, Kubernetes reschedules it elsewhere.
- **Adabas Manager** – a pod that monitors and administers all three Adabas pods. It
  is not published through the Application Gateway.

### Administration

- **Private AKS API server** – the cluster is a private cluster. Administrators and
  the CI/CD agent reach the API server over the private network (ExpressRoute / VPN
  or a self-hosted agent in the spoke).
- **Azure Bastion (Standard SKU)** – in the hub for secure admin access without
  public IPs to any admin VM the customer runs alongside the cluster. Bastion
  connects to VMs, not to pods.
- **Adabas Manager access** – via `kubectl port-forward`, or through an internal load
  balancer reachable only over ExpressRoute / VPN.

### Namespaces and placement

- **`adanat-web`** – the web-facing tier; the only namespace exposed through the
  Application Gateway.
- **`adanat`** – the Natural and Adabas core: Natural Subprogram Server, Natural Web
  I/O Server, Natural Batch, Adabas Manager, and the Adabas cluster StatefulSet. It
  is reachable only from `adanat-web` and from administrators.
- Pods are spread across zones with topology spread constraints and anti-affinity, so
  the loss of a zone leaves the API, online, and database services running.
- Network policies allow only `adanat-web` pods to reach the Natural Services in
  `adanat`, and only Natural pods and Adabas Manager to reach the Adabas pods.

## Key Components on AKS

### Natural OpenAPI Server

- Turns existing Natural subprograms into OpenAPI-described REST endpoints without
  code changes.
- Exposed through its own Service (`natural-openapi`) and gateway endpoint.
- Runs active-active: each API call is handled independently, so every replica can
  serve traffic. Scale with replicas or a Horizontal Pod Autoscaler.
- HTTPS at the Application Gateway and OIDC authentication with Microsoft Entra ID
  or another OIDC provider.

### Natural Subprogram Server

- Runs the Natural subprograms requested through the Natural OpenAPI Server, in its
  own Deployment, separate from interactive Natural workloads.
- One replica per zone behind the `natural-subprogram` Service.
- Natural Security rules apply to subprograms run through the server.

### Natural Availability Server

- Web front end for Natural online applications, running as replicas behind its own
  Service (`natural-availability`) and gateway endpoint.
- Natural session affinity kept in Azure Cache for Redis, so a lost pod or zone does
  not interrupt users.
- Pods can be drained and updated one by one for maintenance without an outage.

### Adabas Cluster (StatefulSet)

- Three pods, one per zone, each with its own persistent volume and independent copy
  of the data – no shared disk.
- SYNC replication between all three pods; automatic failover to a secondary if the
  primary pod, node, or zone is lost.
- Rolling maintenance one pod at a time, and read scale-out on the secondaries.
- Data encrypted at rest by Adabas Encryption for Linux (AEL) and Disk Encryption
  Sets.

## Cloud & Security Resources

### Hub VNet – connectivity & security

- **Azure Application Gateway + WAF (v2, zone-redundant, AGIC)** – ingress, TLS
  termination, and web application firewall, with separate listeners for the Natural
  OpenAPI Server and NAS endpoints.
- **Azure Firewall** – AKS outbound traffic is forced through it with a route table
  (UDR); also controls traffic between hub, spoke, and internet.
- **Virtual Network Gateway** – terminates ExpressRoute / VPN from the customer
  network.
- **DNS zones** – public zone for the gateway host names; private zones for the
  private AKS API server and `privatelink.*` endpoints, linked to both VNets.
- **API Management (optional)** – throttling, versioning, and a developer portal in
  front of the Natural OpenAPI Server.
- **Private Endpoints** – Key Vault, Storage, Container Registry, and Azure Cache for
  Redis.
- **Azure Bastion (Standard SKU)** – secure admin access to VMs without public IPs.
- **VNet peering with gateway transit** – traffic from ExpressRoute / VPN reaches the
  spoke through the hub's Virtual Network Gateway.

### Spoke VNet – AKS

- **Azure Kubernetes Service (AKS)** – private cluster with Azure CNI; system and user
  node pools across three Availability Zones.
- **Kubernetes Services & network policies** – ClusterIP Services between tiers;
  policies as described above.
- **Network Security Groups (NSG)** – on the AKS subnets.

### Shared services

- **Azure Key Vault (CSI Secrets Store)** – TLS certificates, credentials, and licence
  files mounted into pods.
- **Storage Accounts** – Adabas-level backups and file exchange.
- **Azure Cache for Redis (zone-redundant)** – Natural session affinity for NAS.
- **Azure Container Registry** – Software AG container images, pulled through a
  private endpoint.
- **Log Analytics (Container Insights) & Azure Monitor** – metrics, logs, and alerts.
- **Backup Vault (AKS Backup)** – backup of cluster resources and persistent volumes.
  Volume snapshots of a running database are only crash-consistent, so also take
  Adabas-level backups to Storage.
- **Disk Encryption Sets** – customer-managed keys for disk encryption.
- **Managed Disks (Premium SSD, zonal)** – persistent volumes for the Adabas pods.
- **Microsoft Defender for Containers** – runtime and image threat protection.

### Tools & identity

- **IaC (Terraform / Bicep)** – provisions the Azure resources.
- **Azure DevOps (GitOps / Helm)** – deploys the Kubernetes workloads; use a
  self-hosted agent in the spoke, as the API server is private.
- **Microsoft Entra ID (OIDC)** – identity provider for users and API callers.
- **Workload Identity** – least-privilege pod access to Azure services.

## Regional Redundancy (optional)

The diagram covers one region: it protects against the loss of a pod, a node, or an
Availability Zone, not a whole region. Azure Site Recovery does not protect AKS
workloads. Customers who need regional-level redundancy can deploy a second AKS
cluster in a paired region from the same IaC and GitOps definitions, and restore it
from AKS Backup and Adabas backups kept in geo-redundant storage. Agree recovery time
and recovery point objectives with the business.

## Architecture Diagrams

See the `docs` folder:

- `Adabas and Natural on Azure - Advanced AKS v3.0.0.gif` – the current reference
  architecture.
- `Adabas and Natural on Azure - AKS-v3.0.0.gif` – earlier version, kept for
  reference.

Abbreviations: AEL – Adabas Encryption for Linux; NAS – Natural Availability Server;
AGIC – Application Gateway Ingress Controller.

## Intended Use

Production, QA, and advanced development environments needing scalability,
zone-level resilience, API access to Natural business logic, and cloud-native
operations. For non-containerized deployments use the Basic or Resilient HA
blueprints.
