# Adabas & Natural on Azure – Advanced (AKS)

## Overview
The Advanced blueprint runs Adabas & Natural as containers on Azure Kubernetes
Service (AKS) across three Availability Zones. It is intended for scalable,
resilient, cloud-native production workloads. It follows the v3.0.0 Resilient HA
reference architecture, with the virtual machines replaced by Kubernetes workloads.
Customers provision the AKS cluster and supporting resources shown in the diagram and
deploy the official Software AG Adabas and Natural container images.

## What's New in v3.0.0
- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI, deployed as multiple replicas across zones.
- **Natural Subprogram Server** – runs Natural subprograms on behalf of the Natural
  OpenAPI Server, deployed as its own multi-replica Deployment.
- **Natural Availability Server** – multiple replicas sharing session state
  through Azure Cache for Redis.
- **Application Gateway as ingress** – Azure Application Gateway with WAF, managed by
  the Application Gateway Ingress Controller (AGIC), is the single HTTPS entry point,
  with a separate endpoint for each product: one for the Natural OpenAPI Server and
  one for the Natural Availability Server.
- **3-zone Adabas cluster** – a 3-node Adabas cluster with one pod per Availability
  Zone.

## Architecture Layers
- **Ingress (hub VNet):** Azure Application Gateway with WAF receives client HTTPS
  traffic. Through AGIC it publishes two endpoints, each routed to its own Kubernetes
  Service in the cluster:

  | Gateway endpoint (example)       | Kubernetes Service     | Backend pods                  |
  | -------------------------------- | ---------------------- | ----------------------------- |
  | `https://api.adanat.example.com` | `natural-openapi`      | Natural OpenAPI Server        |
  | `https://nas.adanat.example.com` | `natural-availability` | Natural Availability Server   |

  Each endpoint has its own listener, TLS certificate (from Key Vault), WAF policy,
  and health probe, so API traffic and online user traffic can be secured, scaled,
  and monitored independently. Replace the example host names with the customer's
  own DNS names; path-based routing on a single host name (for example `/api` and
  `/nas`) works the same way if separate host names are not wanted.
- **Web tier (`adanat-web` namespace):** Deployments for the **Natural OpenAPI Server**
  and the **Natural Availability Server**, each with at least one replica per zone and
  each behind its own Service. The Natural Availability Server replicas keep session
  state in Azure Cache for Redis, so any replica can serve any user.
- **App tier (`adanat` namespace):** an internal Kubernetes Service (ClusterIP)
  distributes calls from the web tier to:
  - **Natural Subprogram Server** pods (Natural Subprogram Server, Natural Security,
    Natural) in Zones 1 and 2, serving the API calls.
  - **Natural Web/IO** pods (Natural Web/IO, Natural Security, Natural) in Zones 1
    and 2, serving the online applications.
  - an **Adabas Manager** pod in Zone 3 for database administration, reached by
    administrators through Azure Bastion.
- **Data tier (`adanat` namespace):** a 3-replica **StatefulSet** forming the Adabas
  cluster, with one pod per Availability Zone (`adabas-0` primary, `adabas-1` and
  `adabas-2` secondaries). Each pod runs Adabas with AEL and the Adabas REST Server,
  and has its own zonal Premium SSD persistent volume for ASSO/DATA/WORK. The primary
  handles reads and writes over **ADATCP/S** and synchronizes to the secondaries,
  which serve reads.
- **Natural Batch (`adanat` namespace):** runs in its own pods, separate from the
  Adabas pods but placed next to them — one Natural Batch pod per zone, co-located
  with that zone's Adabas pod (pod affinity). The pod in Zone 1 is the **primary**
  and runs batch jobs against the Adabas primary; the pods in Zones 2 and 3 are
  **failover** instances that take over if the primary batch pod or its zone is lost.
  Keeping batch in separate pods lets it be scaled, scheduled, and restarted without
  touching the database pods.

### Namespaces
- **`adanat-web`** – the web-facing tier: Natural OpenAPI Server and Natural
  Availability Server, with the `natural-openapi` and `natural-availability`
  Services. It is the only namespace exposed through the Application Gateway ingress.
- **`adanat`** – the Adabas and Natural core: Natural Subprogram Server, Natural
  Web/IO, Natural Batch, Adabas Manager, and the Adabas cluster StatefulSet. It is
  reachable only from `adanat-web` and from administrators.

Keeping the web tier in its own namespace lets it be scaled, updated, and secured
(RBAC, network policies, resource quotas) separately from the core runtime and
database.

Pods are spread across zones with topology spread constraints and anti-affinity, so
the loss of a zone leaves the API, online, and database services running.

## Key Components on AKS

### Natural OpenAPI Server
- Turns existing Natural subprograms into OpenAPI-described REST endpoints without
  code changes.
- Exposed through its own Service (`natural-openapi`) and gateway endpoint
  (e.g. `https://api.adanat.example.com`).
- Runs active-active: each API call is handled independently, so every replica
  behind the Service can serve traffic. Scale with replicas or a Horizontal Pod
  Autoscaler.
- HTTPS at the Application Gateway and OIDC authentication with Microsoft Entra ID
  or another OIDC provider.

### Natural Subprogram Server
- Runs the Natural subprograms requested through the Natural OpenAPI Server, in its
  own Deployment, separate from interactive Natural workloads.
- Replicas in two zones, reached through an internal Kubernetes Service, so the API
  tier still has a healthy target if a zone is lost. Add a replica in Zone 3 if the
  API workload needs capacity in all three zones.
- Natural Security rules apply to subprograms run through the server.

### Natural Availability Server
- Web front end for Natural online applications, running as multiple replicas behind
  its own Service (`natural-availability`) and gateway endpoint
  (e.g. `https://nas.adanat.example.com`).
- Session and application state held in Azure Cache for Redis, so a lost pod or zone
  does not interrupt users.
- Pods can be drained and updated one by one for maintenance without an outage.

### Adabas Cluster (StatefulSet)
- Three pods, one per zone, each with its own persistent volume and independent copy
  of the data — no shared disk.
- Automatic failover to a secondary if the primary pod, node, or zone is lost.
- Rolling maintenance one pod at a time, and read scale-out on the secondaries.
- Data encrypted at rest by Adabas Encryption for Linux (AEL) and Disk Encryption
  Sets.

## Cloud & Security Resources
- **Azure Kubernetes Service (AKS)** – managed control plane and node pools across
  three Availability Zones.
- **Azure Application Gateway + WAF (AGIC)** – ingress controller, TLS termination,
  and web application firewall, with separate listeners for the Natural OpenAPI
  Server and Natural Availability Server endpoints.
- **Hub & spoke VNets** – connectivity and security services in the hub (Azure
  Firewall, VNet gateways, Bastion, DNS zones, private endpoints, optional API
  Management), the AKS cluster in the peered spoke.
- **Kubernetes Services & Network Policies** – internal ClusterIP services between
  tiers; network policies allow only the `adanat-web` pods to reach the Natural
  services in `adanat`, and only the Natural pods to reach the Adabas pods.
- **Persistent Volumes (Premium SSD, zonal)** – durable storage for each Adabas pod.
- **Azure Cache for Redis** – shared session state for the Natural Availability
  Server.
- **Azure Container Registry** – Software AG container images.
- **Azure Key Vault (CSI Secrets Store)** – TLS certificates, credentials, and
  license files mounted into pods.
- **Microsoft Entra ID & Workload Identity** – OIDC for users and API callers;
  least-privilege pod access to Azure services.
- **Azure Monitor & Log Analytics (Container Insights)** – metrics, logs, and alerts.
- **Backup Vault (AKS Backup)** – backup of cluster resources and persistent volumes.
- **Disk Encryption Sets** – customer-managed keys for disk encryption.
- **Microsoft Defender for Containers** – runtime and image threat protection.
- **ExpressRoute** – private connectivity from the customer network.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on Azure - AKS-v3.0.0.gif`

## Intended Use
Production, QA, and advanced development environments needing scalability,
zone-level resilience, API access to Natural business logic, and cloud-native
operations. For non-containerized deployments use the Basic or Resilient HA
blueprints.
