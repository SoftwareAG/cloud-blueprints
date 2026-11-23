# Adabas & Natural on Azure – Resilient HA

## Overview

The Resilient HA blueprint runs Adabas & Natural on Azure Virtual Machines spread
across three Availability Zones in one region, in a hub-and-spoke network. Every
tier survives the loss of a VM or a whole zone. It is intended for production
workloads. Customers provision the Azure resources shown in the diagram and install
the Adabas (data) and Natural (application) components on those virtual machines.

## What's New in v3.0.0

- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI. In this blueprint it runs active-active in two Availability Zones behind
  the Application Gateway.
- **Natural Subprogram Server** – hosts and runs Natural subprograms on behalf of
  remote callers. It runs on an App VM in each of two zones behind an internal load
  balancer, so API calls survive the loss of a node or a zone.

## Reference Architecture

### Web tier (Web subnet, zones 1 and 2)

- **Natural OpenAPI Server** – one Web VM per zone, active-active.
- **Natural Availability Server (NAS)** – one Web VM per zone. NAS keeps Natural
  session affinity in **Azure Cache for Redis** (zone-redundant), so a session keeps
  working when it lands on the other NAS instance.
- **Ingress:** Azure Application Gateway + WAF (v2, zone-redundant) in the hub VNet
  is the public HTTPS entry point, in front of both services. API calls are
  authenticated with OIDC through Microsoft Entra ID. Internal and admin traffic
  from the customer network comes in privately over ExpressRoute / VPN.

### App tier (App subnet, zones 1 and 2)

- **Internal Load Balancer** (Standard SKU, zone-redundant) – distributes calls from
  the web tier to healthy App VMs in either zone.
- **App VMs** – each runs Natural, Natural Security, and either the **Natural
  Subprogram Server** (behind the Natural OpenAPI Server) or the **Natural Web I/O
  Server** (behind NAS).

### Data tier (Data subnet, zones 1, 2 and 3)

- **Adabas Cluster** – three Data VMs, one per zone, each running Adabas (with AEL)
  and the Adabas Rest Server: one **primary** and two **secondaries**. The cluster
  uses a shared-nothing architecture: every node has its own Premium SSD managed
  disks, and **SYNC replication** runs between all three nodes, so every node holds
  the current data at all times.
- **Access:** the App VMs send all read/write (R/W) calls to the primary over
  **ADATCP/S**; the two secondaries also serve read transactions.
- **Failover:** if the primary fails, a secondary takes over. Natural's Adabas client
  locates the new primary, so Natural reconnects without configuration changes.
- **Natural Batch** – installed on every Data VM but active on one node at a time
  (the primary's VM in the diagram); another node takes over on failover. Each Data VM needs a Natural installation and
  licence for this.

### Management (Mgmt subnet, zone 3)

- **Adabas Manager** – a dedicated VM that monitors and administers all three Adabas
  nodes. It has no public ingress and is not published through the Application
  Gateway.
- **Jumphost** – a VM with admin tools that reaches all workload VMs over SSH.
- **Admin access:** Azure Bastion (Standard SKU) to the Jumphost and to Adabas Manager.
  Administrators on the customer network can also reach Adabas Manager privately by
  its private DNS name over ExpressRoute / VPN.
- Both management VMs are single instances; they are not on the request path, so a
  zone 3 outage does not stop the application.

## Key New Products

### Natural OpenAPI Server

The Natural OpenAPI Server turns existing Natural subprograms into REST APIs without
changes to the Natural code.

- **API-first access:** each exposed subprogram becomes a REST endpoint with an
  OpenAPI specification that client teams can use to generate code and test.
- **Active-active instances:** each API call is handled independently, so every
  instance behind the Application Gateway can serve traffic at the same time.
- **Zone resilience:** instances in two Availability Zones keep the API available if a
  zone is lost.
- **Secure by default:** HTTPS for transport and OIDC for authentication, integrated
  with Microsoft Entra ID or another OIDC provider.
- **Integration ready:** Azure API Management can optionally be added in front for
  throttling, versioning, and a developer portal (not shown in the diagram).

### Natural Subprogram Server

The Natural Subprogram Server is the runtime that runs Natural subprograms requested
by remote callers.

- **Redundant execution:** an instance runs on an App VM in each zone, so the API
  layer always has a healthy target.
- **Isolation from online users:** API-driven calls run in their own server
  processes, separate from interactive Natural sessions.
- **Elastic capacity:** add App VMs or server processes to scale API throughput
  without changing the application.
- **Consistent security:** Natural Security rules apply to subprograms run through the
  server, just as they do for online users.

### Why It Matters

With the Natural OpenAPI Server and the Natural Subprogram Server deployed across
zones, Natural business logic can be offered as highly available APIs alongside the
existing online applications. The same resilient application and data tiers serve
both channels.

## Cloud & Security Resources

### Hub VNet – connectivity & security

- **Azure Application Gateway + WAF (v2, zone-redundant)** – one resource with the Web
  Application Firewall policy attached; TLS termination and the public HTTPS entry
  point for the Natural OpenAPI Server and NAS.
- **Azure Firewall** – controls and logs traffic between the hub, the spoke, and the
  internet.
- **Virtual Network Gateway** – terminates ExpressRoute / VPN from the customer
  network.
- **Azure Bastion (Standard SKU)** – administrative access without public IPs. It sits
  in the hub and reaches the spoke over peering (Basic SKU or higher is required for
  peered VNets; Standard adds native-client and IP-based connections).
- **DNS zones** – `privatelink.*` private DNS zones, linked to both VNets, for private
  endpoints and internal names such as Adabas Manager; a public DNS zone for the
  Application Gateway hostname.
- **Private Endpoints** – private access to Key Vault, Storage, and Azure Cache for
  Redis.
- **VNet peering with gateway transit** – traffic from ExpressRoute / VPN reaches the
  spoke through the hub's Virtual Network Gateway.

### Spoke VNet – workload

- **Azure Virtual Machines (multi-zone)** – Web, App, and Data VMs across zones, plus
  the management VMs in zone 3.
- **Subnets** – Web, App, Data, and Mgmt subnets (subnets span all zones).
- **Internal Load Balancer (Standard, zone-redundant)** – web tier to app tier.
- **Network Security Groups (NSG)** – one per subnet. HTTPS to the web tier only from
  the Application Gateway, app tier only from the web tier, Adabas ports only from
  the app tier and Adabas Manager, Adabas cluster replication ports only between the
  three Data VMs, SSH only from the Jumphost and Bastion.

### Shared services

- **Azure Key Vault** – TLS certificates and secrets for the web tier.
- **Storage Accounts** – backups and file exchange, reached through private endpoints.
- **Azure Cache for Redis (zone-redundant)** – Natural session affinity for NAS.
- **Log Analytics & Azure Monitor** – logs, metrics, and alerts.
- **Recovery Services Vault** – VM backup and restore.
- **Managed Disks (Premium SSD)** – per-node storage for Adabas data and Natural files.
- **Disk Encryption Sets** – encryption at rest with customer-managed keys.
- **Network Watcher** – network diagnostics and flow logs.
- **Microsoft Defender for Cloud** – security posture and threat protection.

### Tools & identity

- **IaC (Terraform / Bicep)** – provisions the resources above.
- **Azure DevOps** – build and release pipelines (use a self-hosted agent in the spoke,
  as the VMs have no public ingress).
- **Microsoft Entra ID (OIDC)** – identity provider for API authentication.
- **Entra Domain Services (LDAP/S)** – directory for OS and Natural Security user
  authentication.
- **Managed Identity** – least-privilege access from VMs to Azure services.

## Regional Redundancy (optional)

The diagram covers one region: it protects against the loss of a VM or an
Availability Zone, not a whole region. Customers who need regional-level redundancy
can add **Azure Site Recovery** to replicate the VMs to a paired region and fail over
the whole stack there. Plan the secondary region's network, DNS, and Application
Gateway alongside it, and agree recovery time and recovery point objectives with the
business.

## Architecture Diagrams

See the `docs` folder:

- `Adabas and Natural on Azure - Resilient HA v3.0.0.gif` – the current reference
  architecture.
- `Adabas and Natural on Azure - Resilient HA - Detailed Security-v3.0.0.gif` –
  earlier detailed-security view, kept for reference.

Abbreviations: AEL – Adabas Encryption for Linux; NAS – Natural Availability Server.

## Intended Use

Production workloads requiring high availability and highly available API access to
Natural business logic, with optional regional redundancy through Azure Site Recovery.
For development and test stages, or production without HA, use the Basic blueprint;
for container-native scaling, use the Advanced blueprint.
