# Adabas & Natural on GCP – Basic

## Overview
The Basic blueprint runs Adabas & Natural on Google Compute Engine within a single
Google Cloud project, VPC, and zone (e.g. `europe-west-3` / `europe-west-3a`). It is
intended for development, prototyping, and small production footprints. Customers
provision the GCP resources shown in the diagrams and install the Adabas (data) and
Natural (application) components on the virtual machine(s).

## Architecture Layers
- **Application layer (Natural):** the Natural runtime and interfaces — for example
  **Natural Online**, **NWO** (Natural Web I/O), **NDV** (Natural Development
  Server), **Natural for Ajax**, **Natural Security**, **Natural Batch**, and the
  **Natural Availability Server**.
- **Data layer (Adabas):** the Adabas database and its services — **Adabas (AEL –
  Adabas Encryption for Linux)**, the **Adabas REST Server**, and **Adabas Manager**.

In the single-instance shapes both layers run on one Compute Engine VM; in the
distributed shape the Natural and Adabas layers run on separate VMs that communicate
over **ADATCP** (read/write).

## Variants
- **Development – Single Instance:** one VM hosting Natural Online, NDV, Adabas (AEL),
  and the Adabas REST Server. Developers connect from **NaturalONE / NJX** over
  SSL/TLS; end users connect via terminal emulation over SSH.
- **Production – Single Instance:** one hardened VM hosting Natural Online, NWO, the
  Natural Availability Server, Adabas (AEL), the Adabas REST Server, and Adabas
  Manager for a small production footprint.
- **Distributed:** the Natural application VM (NWO, Natural Security, Natural Online,
  Natural for Ajax) and the Adabas data VM (Natural Batch, Adabas Manager, Adabas
  REST Server, Adabas AEL) run separately and communicate over ADATCP, allowing each
  layer to be sized, secured, and maintained independently.

## Connectivity & Access
- **Terminal emulation over SSH** – end-user access to Natural.
- **HTTPS / SSL/TLS** – browser and NaturalONE / NJX developer access to the
  application interfaces.
- **Google Cloud Interconnect + Cloud Router** – private connectivity between the
  customer network and the Google Cloud project.

## Cloud & Security Resources
- **Compute Engine** – VM(s) hosting the application and data layers.
- **VPC, Subnet & Zone** – private, isolated network boundary for the workload.
- **Firewall Rules** – layer-scoped, tag-based rules; SSH restricted to trusted
  ranges and Adabas ports reachable only from the Natural layer.
- **Persistent Disk & Filestore** – block and file storage for Adabas data and
  Natural files.
- **Cloud Storage & Cloud Backup** – object storage and backup/restore targets.
- **Cloud Key Management (Cloud KMS)** – customer-managed encryption keys; combined
  with Adabas Encryption for Linux (AEL) for data protection at rest.
- **Cloud DNS & Cloud Monitoring** – name resolution and observability.
- **IAM Service Accounts** – least-privilege access from instances to GCP services.

Customer tooling shown in the diagrams — **Terraform**, **GitOps**, and **OIDC** — is
used to provision infrastructure and manage identity according to your organization's
standards.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on GCP Basic Setup Development Single Instance-v2.0.0.gif`
- `Adabas and Natural on GCP Basic Setup PROD Single Instance.gif`
- `Adabas and Natural on GCP Basic Setup Distributed-v2.0.0.gif`

## Intended Use
Development, prototyping, and small production workloads. For high availability and
fault tolerance, use the Resilient HA blueprint; for container-native scaling, use
the Advanced blueprint.
