# Adabas & Natural on Azure – Resilient HA

## Overview
The Resilient HA blueprint runs Adabas & Natural on Azure across multiple
Availability Zones for high availability and fault tolerance. It is intended for
production workloads. Customers provision the Azure resources shown in the diagrams
and install the Adabas (data) and Natural (application) components on those virtual
machines.

## What's New in v3.0.0
- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI. In this blueprint it runs as multiple instances across Availability
  Zones behind a load balancer.
- **Natural Subprogram Server** – hosts and runs Natural subprograms on behalf of
  remote callers. It is deployed on every Natural VM, so API calls survive the loss of
  a single node or zone.

## Architecture Layers
- **API layer (Natural OpenAPI Server):** multiple instances spread across
  Availability Zones, fronted by a load balancer. Client traffic over **HTTPS** fails
  over automatically between healthy instances.
- **Application layer (Natural):** multiple Natural VMs spread across Availability
  Zones, fronted by a load balancer so client traffic fails over automatically
  between healthy nodes. Each VM also runs the **Natural Subprogram Server**, which
  serves the subprogram calls coming from the API layer.
- **Data layer (Adabas):** Adabas VMs in separate Availability Zones with
  replication/clustering for database resilience, backed by dedicated Managed Disks.

## Key New Products

### Natural OpenAPI Server
The Natural OpenAPI Server turns existing Natural subprograms into REST APIs without
changes to the Natural code.

- **API-first access:** each exposed subprogram becomes a REST endpoint with an
  OpenAPI specification that client teams can use to generate code and test.
- **Active-active instances:** each API call is handled independently, so every
  instance behind the load balancer can serve traffic at the same time.
- **Zone resilience:** instances in at least two Availability Zones keep the API
  available if a zone is lost.
- **Secure by default:** HTTPS for transport and OIDC for authentication, integrated
  with Microsoft Entra ID or another OIDC provider.
- **Integration ready:** can sit behind Azure API Management or Azure Application
  Gateway (with WAF) for rate limiting, versioning, and threat protection.

### Natural Subprogram Server
The Natural Subprogram Server is the runtime that runs Natural subprograms requested
by remote callers.

- **Redundant execution:** an instance runs on each Natural VM, so the API layer
  always has a healthy target in another zone.
- **Isolation from online users:** API-driven calls run in their own server
  processes, separate from interactive Natural sessions.
- **Elastic capacity:** add Natural VMs or server processes to scale API throughput
  without changing the application.
- **Consistent security:** Natural Security rules apply to subprograms run through the
  server, just as they do for online users.

### Why It Matters
With the Natural OpenAPI Server and the Natural Subprogram Server deployed across
zones, Natural business logic can be offered as highly available APIs alongside the
existing online applications. The same resilient application and data layers serve
both channels.

## Cloud & Security Resources
- **Azure Virtual Machines (Multi-Zone)** – API, application, and data nodes
  distributed across Availability Zones.
- **Azure Application Gateway + WAF** – HTTPS entry point in the hub VNet; TLS
  termination and web application firewall in front of the Natural OpenAPI Server and
  Natural Availability Server VMs.
- **Azure Load Balancer (internal)** – distributes calls from the web tier to healthy
  Natural Subprogram Server and Natural Web/IO VMs.
- **Azure Bastion & Jumphost** – administrative access to the Adabas Manager and the
  workload VMs without public IPs.
- **Azure Cache for Redis** – shared session state for the Natural Availability
  Server instances.
- **Azure API Management (optional)** – API gateway, throttling, and developer portal
  in front of the Natural OpenAPI Server.
- **Azure Managed Disks** – encrypted persistent storage for each node.
- **Virtual Network (VNet) & Subnets (Multi-Zone)** – redundant network layout with
  separate subnets per layer.
- **Network Security Groups (NSG)** – layer-scoped rules; HTTPS to the API layer only
  from approved sources, Adabas ports reachable only from the Natural layer,
  management access restricted to trusted ranges.
- **Microsoft Entra ID (OIDC)** – identity provider for API authentication.
- **Azure Key Vault** – TLS certificates and secrets for the API layer.
- **Managed Identity** – least-privilege access to Azure services (backups,
  monitoring, Key Vault).
- **Encryption** – encryption in transit and at rest across the API, application,
  and data layers.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on Azure - Resilient HA - Detailed Security-v3.0.0.gif`

## Intended Use
Production workloads requiring high availability, business continuity, disaster
recovery, and highly available API access to Natural business logic. For simpler
development setups use the Basic blueprint; for container-native scaling use the
Advanced blueprint.
