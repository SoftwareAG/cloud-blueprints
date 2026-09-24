# Adabas & Natural on GCP – Resilient HA

## Overview
The Resilient HA blueprint runs Adabas & Natural on GCP across multiple zones for
high availability and fault tolerance. It is intended for production workloads.
Customers provision the GCP resources shown in the diagram and install the Adabas
(data) and Natural (application) components on those instances.

## Architecture Layers
- **Application layer (Natural):** multiple Natural Compute Engine instances spread
  across zones, fronted by a load balancer so client traffic fails over automatically
  between healthy nodes.
- **Data layer (Adabas):** Adabas Compute Engine nodes in separate zones with
  replication/clustering for database resilience, backed by dedicated Persistent
  Disks (regional disks optional for synchronous replication).

## Cloud & Security Resources
- **Compute Engine (Multi-Zone)** – application and data nodes distributed across
  zones.
- **Cloud Load Balancing** – distributes traffic to healthy Natural nodes.
- **Persistent Disks** – encrypted persistent storage for each node.
- **VPC & Subnets (Regional, Multi-Zone)** – redundant network layout with separate
  subnets per layer.
- **Firewall Rules** – layer-scoped, tag-based rules; Adabas ports reachable only
  from the Natural layer, management access restricted to trusted ranges.
- **IAM Service Accounts** – least-privilege access to GCP services (backups,
  monitoring).
- **Encryption** – encryption in transit and at rest across the application and data
  layers.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on GCP Resilient HA - Detailed Security-v2.0.0.gif`

## Intended Use
Production workloads requiring high availability, business continuity, and disaster
recovery. For simpler development setups use the Basic blueprint; for container-native
scaling use the Advanced blueprint.
