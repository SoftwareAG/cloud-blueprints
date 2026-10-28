# Adabas & Natural on Azure – Basic

## Overview
The Basic blueprint runs Adabas & Natural on Azure Virtual Machines. It is intended
for development, prototyping, and lift-and-shift workloads. Customers provision the
Azure resources shown in the diagrams and install the Adabas (data) and Natural
(application) components on those virtual machines.

## What's New in v3.0.0
- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI, so modern applications, portals, and integration platforms can call
  Natural business logic over HTTPS.
- **Natural Subprogram Server** – hosts and runs Natural subprograms on behalf of
  remote callers such as the Natural OpenAPI Server, separating API traffic from
  interactive Natural sessions.

## Architecture Layers
- **API layer (Natural OpenAPI Server):** publishes selected Natural subprograms as
  OpenAPI-described REST services over **HTTPS**, secured with OIDC.
- **Application layer (Natural):** the Natural runtime and applications, plus the
  **Natural Subprogram Server** that runs the subprograms behind each API call.
  Reachable by end users on the Natural/application ports.
- **Data layer (Adabas):** the Adabas database and its files, held on dedicated
  Azure Managed Disks.

The Basic blueprint supports three shapes:
- **Development – Single Instance:** all layers, including the Natural OpenAPI Server
  and the Natural Subprogram Server, co-located on one VM for simplicity.
- **Distributed:** the Adabas and Natural layers run on separate VMs so each layer
  can be sized, secured, and maintained independently. The Natural OpenAPI Server and
  the Natural Subprogram Server run alongside Natural on the application VM.
- **Production – Single Instance:** a single hardened VM for small production
  footprints, with the API endpoint published only over HTTPS.

## Key New Products

### Natural OpenAPI Server
The Natural OpenAPI Server turns existing Natural subprograms into REST APIs without
changes to the Natural code.

- **API-first access:** each exposed subprogram becomes a REST endpoint with an
  OpenAPI specification that client teams can use to generate code and test.
- **No code changes:** existing business logic stays in Natural. The server maps the
  subprogram parameters to JSON request and response bodies.
- **Secure by default:** HTTPS for transport and OIDC for authentication, integrated
  with Microsoft Entra ID or another OIDC provider.
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
- **Azure Virtual Machines** – compute hosting the API, application, and data layers.
- **Azure Managed Disks** – persistent storage for Adabas data and Natural files;
  enable encryption at rest.
- **Virtual Network (VNet) & Subnets** – private network boundary; place each layer
  in its own subnet where the layers are separated.
- **Network Security Groups (NSG)** – layer-scoped firewalls. Restrict SSH/RDP to
  trusted ranges, allow HTTPS to the Natural OpenAPI Server only from approved
  clients, and allow the Adabas ports only from the Natural layer.
- **Microsoft Entra ID (OIDC)** – identity provider for API authentication.
- **Azure Key Vault** – storage for TLS certificates and secrets used by the Natural
  OpenAPI Server.
- **Managed Identity** – least-privilege access from VMs to Azure services (e.g.
  Blob Storage for backups, Key Vault for certificates).

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on Azure - Basic Development v3.0.0.gif`
- `Adabas and Natural on Azure - Basic Distributed v3.0.0.gif`

## Intended Use
Development, prototyping, API enablement pilots, and small production workloads. For
high availability and fault tolerance, use the Resilient HA blueprint; for
container-native scaling, use the Advanced blueprint.
