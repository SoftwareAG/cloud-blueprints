# Adabas & Natural on GCP – Basic

## Overview
The Basic blueprint runs Adabas & Natural on Google Compute Engine instances. It is
intended for development, prototyping, and lift-and-shift workloads. Customers
provision the GCP resources shown in the diagrams and install the Adabas (data) and
Natural (application) components on those instances.

## Architecture Layers
- **Application layer (Natural):** the Natural runtime and applications. Reachable
  by end users on the Natural/application ports.
- **Data layer (Adabas):** the Adabas database and its files, held on dedicated
  Persistent Disks.

The Basic blueprint supports:
- **Development – Single Instance:** both layers co-located on one Compute Engine
  instance for simplicity.
- **Distributed:** the Adabas and Natural layers run on separate Compute Engine
  instances so each layer can be sized, secured, and maintained independently.

## Cloud & Security Resources
- **Compute Engine** – compute hosting the application and data layers.
- **Persistent Disks** – persistent storage for Adabas data and Natural files;
  enable encryption at rest.
- **VPC & Subnets** – private network boundary; place each layer in its own subnet
  where the layers are separated.
- **Firewall Rules** – layer-scoped, tag-based firewalls. Restrict SSH to trusted
  ranges and allow the Adabas ports only from the Natural layer.
- **IAM Service Accounts** – least-privilege access from instances to GCP services
  (e.g. Cloud Storage for backups).

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on GCP Basic Setup Development Single Instance-v2.0.0.gif`
- `Adabas and Natural on GCP Basic Setup Distributed-v2.0.0.gif`

## Intended Use
Development, prototyping, and small production workloads. For high availability and
fault tolerance, use the Resilient HA blueprint; for container-native scaling, use
the Advanced blueprint.
