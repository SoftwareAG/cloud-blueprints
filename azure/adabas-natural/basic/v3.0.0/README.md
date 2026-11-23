# Adabas & Natural on Azure – Basic

## Overview

The Basic blueprint runs Adabas & Natural on Azure Virtual Machines in a hub-and-spoke
network. It covers development, prototyping, test stages (INT, UAT, QA), and
lift-and-shift workloads that do not need high availability. Customers provision the
Azure resources shown in the diagrams and install the Adabas (data) and Natural
(application) components on those virtual machines.

## What's New in v3.0.0

- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI, so modern applications, portals, and integration platforms can call
  Natural business logic over HTTPS.
- **Natural Subprogram Server** – hosts and runs Natural subprograms on behalf of
  remote callers such as the Natural OpenAPI Server, separating API traffic from
  interactive Natural sessions.

## Reference Architectures

The Basic blueprint has two variants. Both use the same hub-and-spoke network and the
same shared services; they differ in how many VMs run the workload and how users reach
it.

### Development – Single VM

A workspace for developers and internal teams. Every layer runs on one VM.

- **Development VM:** Natural OpenAPI Server, Natural Subprogram Server, Natural, and
  the Natural Development Server (NDV) alongside Adabas (with AEL), the Adabas Rest
  Server, and Adabas Manager. Adabas data sits on Standard SSD managed disks, which
  are enough for development I/O at lower cost.
- **No public ingress:** there is no Application Gateway. Users call the Natural
  OpenAPI Server over HTTPS, developers use terminal emulation over SSH and NaturalONE
  (NJX, VS Code) over SSL/TLS to NDV – all privately through ExpressRoute / VPN.
- **No OIDC:** API access is limited by the private network and NSG rules.
- **Admin access:** Azure Bastion to the VM.
- Not for production.

### Distributed – Application VM and Data VM

The Adabas and Natural layers run on separate VMs, so each can be sized, secured, and
maintained on its own. Suited to INT, UAT, and QA, or to production where HA is not
required.

- **Application VM:** Natural OpenAPI Server with the Natural Subprogram Server behind
  it; Natural Availability Server (NAS) with the Natural Web I/O Server behind it;
  Natural Security and the Natural runtime underneath both.
- **Data VM:** Adabas (with AEL), Adabas Rest Server, Adabas Manager, and Natural Batch.
  Adabas data sits on Premium SSD managed disks. Natural Batch needs its own Natural
  installation and licence on this VM.
- **Public ingress:** Azure Application Gateway (WAF_v2) in the hub VNet is the public
  HTTPS entry point, in front of the Natural OpenAPI Server and NAS. API calls are
  authenticated with OIDC through Microsoft Entra ID.
- **Private traffic:** internal and admin traffic comes in through ExpressRoute / VPN.
- **Application to data:** the Application VM reads and writes Adabas over
  **ADATCP/S**.
- **Admin access:** Azure Bastion to both VMs – the Application VM for patching,
  Natural deployment, and NAS / Web I/O configuration; the Data VM for Adabas Manager.
- **Single instance:** every layer runs once. NAS keeps session state in memory, so no
  external session store is needed. Each VM sits in its own subnet with a layer-scoped
  NSG.

## Key New Products

### Natural OpenAPI Server

The Natural OpenAPI Server turns existing Natural subprograms into REST APIs without
changes to the Natural code.

- **API-first access:** each exposed subprogram becomes a REST endpoint with an
  OpenAPI specification that client teams can use to generate code and test.
- **No code changes:** existing business logic stays in Natural. The server maps the
  subprogram parameters to JSON request and response bodies.
- **Secure transport and identity:** HTTPS for transport; supports OIDC for
  authentication with Microsoft Entra ID or another OIDC provider (used in the
  Distributed variant).
- **Integration ready:** can sit behind Azure API Management or another gateway for
  rate limiting, versioning, and a developer portal.

### Natural Subprogram Server

The Natural Subprogram Server is the runtime that runs Natural subprograms requested
by remote callers.

- **Dedicated execution:** runs API-driven subprogram calls in their own server
  processes, separate from interactive Natural sessions.
- **Direct data access:** runs on the Natural application layer and reaches Adabas
  over **ADATCP/S**, like any other Natural workload.
- **Independent sizing:** the number of server processes can be tuned to API demand
  without affecting online users.

## Cloud & Security Resources

### Hub VNet – connectivity & security

- **Azure Application Gateway (WAF_v2)** – Distributed only. One resource with the Web
  Application Firewall policy attached; the public HTTPS entry point in front of the
  Natural OpenAPI Server and NAS.
- **Azure Firewall** – controls and logs traffic between the hub, the spoke, and the
  internet.
- **Virtual Network Gateway** – terminates ExpressRoute / VPN from the customer
  network.
- **Azure Bastion** – administrative access without public IPs on the VMs. Bastion sits
  in the hub and reaches the spoke over peering, so use the Basic SKU or higher; the
  Developer SKU does not support peered VNets.
- **DNS zones** – `privatelink.*` private DNS zones, linked to both VNets, for the
  private endpoints. The Distributed variant also needs a public DNS zone for the
  Application Gateway hostname.
- **Private Endpoints** – private access to Storage and Key Vault from the VNets.
- **VNet peering with gateway transit** – traffic from ExpressRoute / VPN reaches the
  spoke through the hub's Virtual Network Gateway.

### Spoke VNet – workload

- **Azure Virtual Machines** – one VM (Development) or an Application VM and a Data VM
  (Distributed), in one availability zone.
- **Network Security Groups (NSG)** – layer-scoped firewalls. Restrict SSH/RDP to
  trusted ranges, allow HTTPS to the Natural OpenAPI Server and NAS only from the
  Application Gateway or approved clients, and allow the Adabas ports only from the
  Natural layer.

### Shared services

- **Azure Key Vault** – TLS certificates and secrets for the Natural OpenAPI Server and
  NAS.
- **Storage Accounts** – backups and file exchange, reached through private endpoints.
- **Log Analytics & Azure Monitor** – logs, metrics, and alerts for the VMs and
  network services.
- **Recovery Services Vault** – VM backup and restore.
- **Managed Disks** – persistent storage for Adabas data and Natural files: Standard
  SSD in the Development variant, Premium SSD in the Distributed variant.
- **Disk Encryption Sets** – encryption at rest with customer-managed keys.
- **Network Watcher** – Distributed only; network diagnostics and flow logs.
- **Microsoft Defender for Cloud** – security posture and threat protection.

### Tools & identity

- **IaC (Terraform / Bicep)** – provisions the resources above.
- **Azure DevOps** – Distributed only; build and release pipelines. With no public
  ingress to the VMs, use a self-hosted agent in the spoke.
- **Microsoft Entra ID (OIDC)** – Distributed only; identity provider for API
  authentication.
- **Entra Domain Services (LDAP/S)** – directory for OS and Natural Security user
  authentication.
- **Managed Identity** – least-privilege access from VMs to Azure services (e.g. Blob
  Storage for backups, Key Vault for certificates).

## Architecture Diagrams

See the `docs` folder:

- `Adabas and Natural on Azure - Basic Development v3.0.0.gif` – single VM, private
  access only.
- `Adabas and Natural on Azure - Basic Distributed v3.0.0.gif` – Application VM and
  Data VM, public HTTPS through Application Gateway (WAF_v2).

Abbreviations: AEL – Adabas Encryption for Linux; NDV – Natural Development Server;
NAS – Natural Availability Server.

## Intended Use

| Variant      | Typical use                                        | Public ingress | HA |
|--------------|----------------------------------------------------|----------------|----|
| Development  | Developer workspaces, prototyping, API pilots      | No             | No |
| Distributed  | INT, UAT, QA; production where HA is not required  | Yes (WAF_v2)   | No |

For high availability and fault tolerance, use the Resilient HA blueprint; for
container-native scaling, use the Advanced blueprint.
