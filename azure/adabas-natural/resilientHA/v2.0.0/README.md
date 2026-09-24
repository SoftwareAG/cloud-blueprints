# Adabas & Natural on Azure – Resilient HA

## Overview
The Resilient HA blueprint runs Adabas & Natural on Azure across multiple
Availability Zones for high availability and fault tolerance. It is intended for
production workloads. Customers provision the Azure resources shown in the diagrams
and install the Adabas (data) and Natural (application) components on those virtual
machines.

## Architecture Layers
- **Application layer (Natural):** multiple Natural VMs spread across Availability
  Zones, fronted by a load balancer so client traffic fails over automatically
  between healthy nodes.
- **Data layer (Adabas):** Adabas VMs in separate Availability Zones with
  replication/clustering for database resilience, backed by dedicated Managed Disks.

## Cloud & Security Resources
- **Azure Virtual Machines (Multi-Zone)** – application and data nodes distributed
  across Availability Zones.
- **Azure Load Balancer** – distributes traffic to healthy Natural nodes.
- **Azure Managed Disks** – encrypted persistent storage for each node.
- **Virtual Network (VNet) & Subnets (Multi-Zone)** – redundant network layout with
  separate subnets per layer.
- **Network Security Groups (NSG)** – layer-scoped rules; Adabas ports reachable only
  from the Natural layer, management access restricted to trusted ranges.
- **Managed Identity** – least-privilege access to Azure services (backups,
  monitoring).
- **Encryption** – encryption in transit and at rest across the application and data
  layers.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on Azure - Resilient HA-v2.0.0.gif`
- `Adabas and Natural on Azure - Resilient HA - Detailed Security-v2.0.0.gif`

## Intended Use
Production workloads requiring high availability, business continuity, and disaster
recovery. For simpler development setups use the Basic blueprint; for container-native
scaling use the Advanced blueprint.
