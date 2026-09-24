# Adabas & Natural on Azure – Basic

## Overview
The Basic blueprint runs Adabas & Natural on Azure Virtual Machines. It is intended
for development, prototyping, and lift-and-shift workloads. Customers provision the
Azure resources shown in the diagrams and install the Adabas (data) and Natural
(application) components on those virtual machines.

## Architecture Layers
- **Application layer (Natural):** the Natural runtime and applications. Reachable
  by end users on the Natural/application ports.
- **Data layer (Adabas):** the Adabas database and its files, held on dedicated
  Azure Managed Disks.

The Basic blueprint supports three shapes:
- **Development – Single Instance:** both layers co-located on one VM for simplicity.
- **Distributed:** the Adabas and Natural layers run on separate VMs so each layer
  can be sized, secured, and maintained independently.
- **Production – Single Instance:** a single hardened VM for small production
  footprints.

## Cloud & Security Resources
- **Azure Virtual Machines** – compute hosting the application and data layers.
- **Azure Managed Disks** – persistent storage for Adabas data and Natural files;
  enable encryption at rest.
- **Virtual Network (VNet) & Subnets** – private network boundary; place each layer
  in its own subnet where the layers are separated.
- **Network Security Groups (NSG)** – layer-scoped firewalls. Restrict SSH/RDP to
  trusted ranges and allow the Adabas ports only from the Natural layer.
- **Managed Identity** – least-privilege access from VMs to Azure services (e.g.
  Blob Storage for backups).

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on Azure - Basic Setup Development Single Instance-v2.0.0.gif`
- `Adabas and Natural on Azure - Basic Setup Distributed-v2.0.0.gif`
- `Adabas and Natural on Azure - Basic Setup Prod Single Instance-v2.0.0.gif`

## Intended Use
Development, prototyping, and small production workloads. For high availability and
fault tolerance, use the Resilient HA blueprint; for container-native scaling, use
the Advanced blueprint.
