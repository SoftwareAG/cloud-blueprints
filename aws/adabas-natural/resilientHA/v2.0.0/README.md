# Adabas & Natural on AWS – Resilient HA

## Overview
The Resilient HA blueprint runs Adabas & Natural on AWS across multiple Availability
Zones for high availability and fault tolerance. It is intended for production
workloads. Customers provision the AWS resources shown in the diagrams and install
the Adabas (data) and Natural (application) components on those instances.

## Architecture Layers
- **Application layer (Natural):** multiple Natural EC2 instances spread across
  Availability Zones, fronted by a load balancer so client traffic fails over
  automatically between healthy nodes.
- **Data layer (Adabas):** Adabas EC2 nodes in separate Availability Zones with
  replication/clustering for database resilience, backed by dedicated EBS volumes.

## Cloud & Security Resources
- **Amazon EC2 (Multi-AZ)** – application and data nodes distributed across AZs.
- **Elastic Load Balancer** – distributes traffic to healthy Natural nodes.
- **Amazon EBS** – encrypted persistent storage for each node.
- **VPC & Subnets (Multi-AZ)** – redundant network layout with separate subnets per
  layer and per AZ.
- **Security Groups** – layer-scoped rules; Adabas ports reachable only from the
  Natural layer, management access restricted to trusted ranges.
- **IAM Roles** – least-privilege access to AWS services (backups, monitoring).
- **Encryption** – encryption in transit and at rest across the application and data
  layers.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on AWS Resilient HA Setup Detailed_v2.0.0.gif`
- `Adabas and Natural on AWS Resilient HA Setup Detailed_Security&Encryption_v2.0.0.gif`

## Intended Use
Production workloads requiring high availability, business continuity, and disaster
recovery. For simpler development setups use the Basic blueprint; for container-native
scaling use the Advanced blueprint.
