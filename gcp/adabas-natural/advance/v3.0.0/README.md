# Adabas & Natural on GCP – Advanced (GKE)

## Overview
The Advanced blueprint runs Adabas & Natural as containers on Google Kubernetes
Engine (GKE) across three zones of one region. It is intended for scalable, resilient,
cloud-native production workloads. It follows the v3.0.0 Resilient HA reference
architecture, with the Compute Engine VMs replaced by Kubernetes workloads. Customers
provision the GKE cluster and supporting resources shown in the diagram and deploy the
official Software AG Adabas and Natural container images.

## What's New in v3.0.0
- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI, deployed as multiple replicas across zones.
- **Natural Subprogram Server** – runs Natural subprograms on behalf of the Natural
  OpenAPI Server, deployed as its own multi-replica Deployment.
- **Natural Availability Server** – multiple replicas sharing session state through
  Memorystore.
- **Internal Application Load Balancer as ingress** – created by the GKE Gateway
  controller (`gke-l7-rilb` GatewayClass), with a Cloud Armor policy, as the single
  HTTPS entry point, with a separate endpoint for each product: one for the Natural
  OpenAPI Server and one for the Natural Availability Server.
- **3-zone Adabas cluster** – a 3-node Adabas cluster with one pod per zone.
- **Shared VPC network** – a host project owns the VPC network, subnets, firewall
  policies, Cloud NAT, and hybrid connectivity; the GKE cluster runs in a separate
  service project.

## Network Layout
- **Customer access:** client traffic arrives from the customer network over Cloud
  Interconnect or HA VPN, terminated on a Cloud Router in the host project. The load
  balancer is internal; no node or pod has an external IP address.
- **Host project (Shared VPC):** the proxy-only subnet required by the internal
  Application Load Balancer, Cloud NGFW firewall policies, Cloud Router with HA VPN /
  Interconnect attachments, Cloud DNS private zones, Private Service Connect endpoints
  for Google APIs, Cloud NAT, and an admin VM reached through IAP for cluster
  administration.
- **Service project:** a **private, regional GKE cluster** (VPC-native) on a Shared VPC
  subnet, with secondary ranges for pods and Services. The GKE Gateway controller
  creates the internal Application Load Balancer in the service project, using an IP
  address from the Shared VPC subnet. The cluster has private nodes and a private
  control-plane endpoint.

Because the cluster is in a service project, grant the GKE service agent of the
service project the *Host Service Agent User* role and the *Compute Network User* role
on the shared subnet, and create the firewall rules for the load balancer health
checks in the host project.

## Architecture Layers
- **Ingress (service project):** an internal Application Load Balancer with Cloud
  Armor receives client HTTPS traffic. Through a Kubernetes Gateway and two HTTPRoutes
  it publishes two endpoints, each routed to its own Kubernetes Service in the cluster
  (container-native load balancing to pod IPs):

  | Gateway listener (example)       | Kubernetes Service     | Backend pods                  |
  | -------------------------------- | ---------------------- | ----------------------------- |
  | `https://api.adanat.example.com` | `natural-openapi`      | Natural OpenAPI Server        |
  | `https://nas.adanat.example.com` | `natural-availability` | Natural Availability Server   |

  Each endpoint has its own TLS certificate (from Certificate Manager), Cloud Armor
  policy (`GCPBackendPolicy`), and health check (`HealthCheckPolicy`), so API traffic
  and online user traffic can be secured, scaled, and monitored independently. Replace
  the example host names with the customer's own DNS names; path-based routing on a
  single host name (for example `/api` and `/nas`) works the same way.
- **Web tier (`adanat-web` namespace):** Deployments for the **Natural OpenAPI Server**
  and the **Natural Availability Server**, each with at least one replica per zone and
  each behind its own Service. The Natural Availability Server replicas keep session
  state in Memorystore, so any replica can serve any user.
- **App tier (`adanat` namespace):** an internal Kubernetes Service (ClusterIP)
  distributes calls from the web tier to:
  - **Natural Subprogram Server** pods (Natural Subprogram Server, Natural Security,
    Natural) in zones `europe-west3-a` and `-b`, serving the API calls.
  - **Natural Web/IO** pods (Natural Web/IO, Natural Security, Natural) in zones `-a`
    and `-b`, serving the online applications.
  - an **Adabas Manager** pod in zone `-c` for database administration. Administrators
    connect with IAP TCP forwarding to the admin VM and reach the pod through the
    private GKE control-plane endpoint (`kubectl port-forward`).
- **Data tier (`adanat` namespace):** a 3-replica **StatefulSet** forming the Adabas
  cluster, with one pod per zone (`adabas-0` primary, `adabas-1` and `adabas-2`
  secondaries). Each pod runs Adabas with AEL and the Adabas REST Server, and has its
  own zonal Hyperdisk Balanced persistent volume for ASSO/DATA/WORK, provisioned by the
  Compute Engine Persistent Disk CSI driver with a CMEK-enabled StorageClass. The
  primary handles reads and writes over **ADATCP/S** and synchronizes to the
  secondaries, which serve reads.
- **Natural Batch (`adanat` namespace):** runs in its own pods, separate from the
  Adabas pods but placed next to them — one Natural Batch pod per zone, co-located
  with that zone's Adabas pod (pod affinity). The pod in zone `-a` is the **primary**
  and runs batch jobs against the Adabas primary; the pods in zones `-b` and `-c` are
  **failover** instances that take over if the primary batch pod or its zone is lost.
  Keeping batch in separate pods lets it be scaled, scheduled, and restarted without
  touching the database pods.

### Namespaces
- **`adanat-web`** – the web-facing tier: Natural OpenAPI Server and Natural
  Availability Server, with the `natural-openapi` and `natural-availability`
  Services. It is the only namespace exposed through the Gateway.
- **`adanat`** – the Adabas and Natural core: Natural Subprogram Server, Natural
  Web/IO, Natural Batch, Adabas Manager, and the Adabas cluster StatefulSet. It is
  reachable only from `adanat-web` and from administrators.

Keeping the web tier in its own namespace lets it be scaled, updated, and secured
(RBAC, network policies, resource quotas) separately from the core runtime and
database.

Pods are spread across zones with topology spread constraints and anti-affinity, so
the loss of a zone leaves the API, online, and database services running. Because
Hyperdisk and Persistent Disk volumes are zonal, each Adabas pod stays in its own zone
and node pools must have capacity in all three zones (use a StorageClass with
`volumeBindingMode: WaitForFirstConsumer`).

## Key Components on GKE

### Natural OpenAPI Server
- Turns existing Natural subprograms into OpenAPI-described REST endpoints without
  code changes.
- Exposed through its own Service (`natural-openapi`) and Gateway endpoint
  (e.g. `https://api.adanat.example.com`).
- Runs active-active: each API call is handled independently, so every replica
  behind the Service can serve traffic. Scale with replicas or a Horizontal Pod
  Autoscaler.
- HTTPS at the internal Application Load Balancer and OIDC authentication with Cloud
  Identity, Workforce Identity Federation, or another OIDC provider.

### Natural Subprogram Server
- Runs the Natural subprograms requested through the Natural OpenAPI Server, in its
  own Deployment, separate from interactive Natural workloads.
- Replicas in two zones, reached through an internal Kubernetes Service, so the API
  tier still has a healthy target if a zone is lost. Add a replica in zone `-c` if the
  API workload needs capacity in all three zones.
- Natural Security rules apply to subprograms run through the server.

### Natural Availability Server
- Web front end for Natural online applications, running as multiple replicas behind
  its own Service (`natural-availability`) and Gateway endpoint
  (e.g. `https://nas.adanat.example.com`).
- Session and application state held in Memorystore, so a lost pod or zone does not
  interrupt users.
- Pods can be drained and updated one by one for maintenance without an outage.

### Adabas Cluster (StatefulSet)
- Three pods, one per zone, each with its own persistent volume and independent copy
  of the data — no shared disk.
- Automatic failover to a secondary if the primary pod, node, or zone is lost.
- Rolling maintenance one pod at a time, and read scale-out on the secondaries.
- Data encrypted at rest by Adabas Encryption for Linux (AEL) and Hyperdisk encryption
  with Cloud KMS customer-managed keys.

## Cloud & Security Resources
- **Google Kubernetes Engine (GKE Standard, regional, private)** – managed control
  plane and node pools across three zones.
- **Internal Application Load Balancer + Cloud Armor (GKE Gateway)** – ingress, TLS
  termination, and a regional security policy, with separate listeners for the
  Natural OpenAPI Server and Natural Availability Server endpoints. Requires a
  proxy-only subnet in the region.
- **Shared VPC (host & service projects)** – connectivity and security services in the
  host project (Cloud NGFW, Cloud Router with HA VPN / Interconnect, Cloud DNS, Private
  Service Connect, Cloud NAT, admin VM), the GKE cluster and its ingress in the
  service project.
- **Private GKE control-plane endpoint** – cluster administration from the admin VM
  only; no public control-plane access.
- **Firewall Rules & Kubernetes Network Policies** – firewall rules for nodes and load
  balancer health checks; network policies (GKE Dataplane V2) allow only the
  `adanat-web` pods to reach the Natural services in `adanat`, and only the Natural
  pods to reach the Adabas pods.
- **Persistent Volumes (Hyperdisk Balanced, zonal)** – durable storage for each
  Adabas pod through the Persistent Disk CSI driver.
- **Memorystore (Redis / Valkey)** – shared session state for the Natural Availability
  Server.
- **Artifact Registry** – Software AG container images.
- **Secret Manager (Secrets Store CSI driver) & Certificate Manager** – TLS
  certificates, credentials, and license files mounted into pods.
- **Cloud Identity (OIDC) & Workload Identity Federation for GKE** – OIDC for users and
  API callers; least-privilege pod access to Google Cloud services without
  service-account keys.
- **Cloud Logging & Cloud Monitoring** – metrics, logs, and alerts for the cluster and
  workloads.
- **Backup for GKE** – backup of cluster resources and persistent volumes.
- **Cloud KMS** – customer-managed keys for disks, Kubernetes Secrets (application-
  layer secrets encryption), and backups.
- **Security Command Center (GKE threat detection)** – runtime and audit-log threat
  detection.
- **Cloud Interconnect / HA VPN** – private connectivity from the customer network.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on GCP - GKE-v3.0.0.gif`

## Intended Use
Production, QA, and advanced development environments needing scalability,
zone-level resilience, API access to Natural business logic, and cloud-native
operations. For non-containerized deployments use the Basic or Resilient HA
blueprints.
