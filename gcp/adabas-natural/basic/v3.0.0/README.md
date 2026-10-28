# Adabas & Natural on GCP – Basic

## Overview
The Basic blueprint runs Adabas & Natural on Google Compute Engine virtual machines. It
is intended for development, prototyping, and lift-and-shift workloads. Customers
provision the Google Cloud resources shown in the diagrams and install the Adabas
(data) and Natural (application) components on those virtual machines.

## What's New in v3.0.0
- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI, so modern applications, portals, and integration platforms can call
  Natural business logic over HTTPS.
- **Natural Subprogram Server** – hosts and runs Natural subprograms on behalf of
  remote callers such as the Natural OpenAPI Server, separating API traffic from
  interactive Natural sessions.
- **Shared VPC network** – a host project owns the VPC network, subnets, firewall
  policies, Cloud NAT, and hybrid connectivity; the Adabas & Natural VMs run in a
  separate service project attached to it.

## Network Layout
- **Customer access:** all client traffic (HTTPS, and SSH / SSL/TLS for developers in
  the Development shape) arrives from the customer network over Cloud Interconnect or
  HA VPN, terminated on a Cloud Router in the host project. Nothing is exposed to the
  internet: the load balancer is internal and the VMs have no external IP addresses.
- **Host project (Shared VPC):** the regional internal Application Load Balancer with
  a Cloud Armor security policy, Cloud NGFW firewall policies, Cloud Router with HA VPN
  / Interconnect attachments, Cloud DNS private zones, Private Service Connect
  endpoints for Google APIs, and Cloud NAT for outbound OS and package updates. The
  host project also holds the **proxy-only subnet** that the internal Application Load
  Balancer needs in the region.
- **Service project:** the Compute Engine VMs, placed in Shared VPC subnets – one
  subnet for the Development VM, or separate app and data subnets in the Distributed
  shape – with layer-scoped firewall rules (network tags or secure tags).

The internal Application Load Balancer's forwarding rule uses an IP address from a
Shared VPC subnet; its backend service and instance group stay in the service project
and are referenced across projects (cross-project service referencing, Compute Load
Balancer Services User role). The load balancer can equally be created in the service
project – the diagram shows it with the other shared ingress components.

## Architecture Layers
- **API layer (Natural OpenAPI Server):** publishes selected Natural subprograms as
  OpenAPI-described REST services over **HTTPS**, secured with OIDC. Client traffic
  reaches it through the internal Application Load Balancer and Cloud Armor.
- **Application layer (Natural):** the Natural runtime and applications, plus the
  **Natural Subprogram Server** that runs the subprograms behind each API call.
  Reachable by end users on the Natural/application ports.
- **Data layer (Adabas):** the Adabas database and its files, held on dedicated
  Hyperdisk Balanced (or Persistent Disk SSD) volumes encrypted with Cloud KMS keys.

The Basic blueprint supports three shapes:
- **Development – Single Instance:** all layers, including the Natural OpenAPI Server
  and the Natural Subprogram Server, co-located on one VM for simplicity.
- **Distributed:** the Adabas and Natural layers run on separate VMs, each in its own
  Shared VPC subnet, so each layer can be sized, secured, and maintained
  independently. The Natural OpenAPI Server and the Natural Subprogram Server run
  alongside Natural on the application VM.
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
  with Cloud Identity, Workforce Identity Federation, or another OIDC provider.
- **Integration ready:** can sit behind an API management gateway of the customer's
  choice for rate limiting, versioning, and a developer portal.

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
- **Compute Engine** – VMs hosting the API, application, and data layers.
- **Hyperdisk Balanced / Persistent Disk** – persistent storage for Adabas data and
  Natural files; encrypted at rest with **Cloud KMS** customer-managed keys (CMEK).
- **Shared VPC** – private network boundary; the host project owns the network and
  shares its subnets with the service project that runs the workload.
- **Internal Application Load Balancer + Cloud Armor** – HTTPS entry point and
  regional security policy in front of the Natural OpenAPI Server.
- **Cloud NGFW & Firewall Rules** – layer-scoped firewalls. Allow HTTPS to the Natural
  OpenAPI Server only from the load balancer's proxy-only subnet and health-check
  ranges, and the Adabas ports only from the Natural layer. SSH and NDV (SSL/TLS)
  ports are opened only to developer ranges in the customer network, and only in the
  Development shape; administrators use IAP instead of direct SSH.
- **Identity-Aware Proxy (IAP) TCP forwarding** – administrative SSH access to the VMs
  without bastion hosts, open SSH ports to the network, or external IPs (allow the
  IAP range `35.235.240.0/20` on port 22).
- **Cloud NAT & Private Service Connect** – outbound updates without external IPs;
  private access to Google APIs (Cloud Storage, Secret Manager, Cloud KMS).
- **Cloud Identity (OIDC)** – identity provider for API authentication; **Managed
  Service for Microsoft Active Directory** for LDAP/S where Natural Security uses it.
- **Secret Manager & Certificate Manager** – TLS certificates and secrets used by the
  Natural OpenAPI Server and the load balancer.
- **Service Accounts (IAM)** – least-privilege access from VMs to Google Cloud
  services (e.g. Cloud Storage for backups, Secret Manager for certificates).
- **Cloud Interconnect / HA VPN + Cloud Router** – private connectivity from the
  customer network into the Shared VPC.
- **Cloud Logging, Cloud Monitoring, Backup and DR, Security Command Center, VPC Flow
  Logs** – logging, monitoring, backup, threat detection, and network logging.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on GCP - Basic Development v3.0.0.gif`
- `Adabas and Natural on GCP - Basic Distributed v3.0.0.gif`

## Intended Use
Development, prototyping, API enablement pilots, and small production workloads. For
high availability and fault tolerance, use the Resilient HA blueprint; for
container-native scaling, use the Advanced blueprint.
