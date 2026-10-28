# Adabas & Natural on GCP – Resilient HA

## Overview
The Resilient HA blueprint runs Adabas & Natural on Google Cloud across three zones of
one region for high availability and fault tolerance. It is intended for production
workloads. Customers provision the Google Cloud resources shown in the diagram and
install the Adabas (data) and Natural (application) components on those Compute
Engine VMs.

## What's New in v3.0.0
- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI. In this blueprint it runs as multiple VMs across zones behind an internal
  Application Load Balancer.
- **Natural Subprogram Server** – hosts and runs Natural subprograms on behalf of
  remote callers. It runs on its own VMs in two zones, so API calls survive the loss of
  a single VM or zone.
- **Natural Availability Server** – multiple VMs sharing session state through
  Memorystore.
- **Separate endpoints per product** – the load balancer publishes one host name for
  the Natural OpenAPI Server and one for the Natural Availability Server, each with its
  own backend service.
- **Shared VPC network** – a host project owns the VPC network, subnets, firewall
  policies, Cloud NAT, and hybrid connectivity; the workload runs in a separate
  service project.

## Network Layout
- **Customer access:** client traffic arrives from the customer network over Cloud
  Interconnect or HA VPN, terminated on a Cloud Router in the host project. The load
  balancer is internal; no VM has an external IP address.
- **Host project (Shared VPC):** the regional internal Application Load Balancer with
  a Cloud Armor security policy, the proxy-only subnet it requires, Cloud NGFW firewall
  policies, Cloud Router with HA VPN / Interconnect attachments, Cloud DNS private
  zones, Private Service Connect endpoints for Google APIs, and Cloud NAT.
- **Service project:** three tiers of Shared VPC subnets – web, app, and data. Subnets
  are regional in Google Cloud, so one subnet per tier spans all three zones; each tier
  is protected by its own firewall rules (network tags or secure tags).

## Architecture Layers
- **Ingress (host project):** an internal Application Load Balancer with Cloud Armor
  receives client HTTPS traffic and routes it by host name to two backend services in
  the service project (cross-project service referencing):

  | Load balancer rule (example)     | Backend service        | Backends                      |
  | -------------------------------- | ---------------------- | ----------------------------- |
  | `https://api.adanat.example.com` | `natural-openapi`      | Natural OpenAPI Server        |
  | `https://nas.adanat.example.com` | `natural-availability` | Natural Availability Server   |

  Each rule has its own TLS certificate (from Certificate Manager), Cloud Armor rules,
  and health check, so API traffic and online user traffic can be secured, scaled, and
  monitored independently. Each backend service uses a regional managed instance group
  spread across the three zones. Replace the example host names with the customer's
  own DNS names; path-based rules on a single host name (for example `/api` and `/nas`)
  work the same way.
- **Web tier (web subnet):** **Natural OpenAPI Server** and **Natural Availability
  Server** VMs in all three zones. The Natural Availability Server VMs keep session
  state in Memorystore, so any VM can serve any user.
- **App tier (app subnet):** an internal passthrough Network Load Balancer distributes
  calls from the web tier to:
  - **Natural Subprogram Server** VMs (Natural Subprogram Server, Natural Security,
    Natural) in zones `europe-west3-a` and `-b`, serving the API calls.
  - **Natural Web/IO** VMs (Natural Web/IO, Natural Security, Natural) in zones `-a`
    and `-b`, serving the online applications.
  - an **Adabas Manager** VM in zone `-c` for database administration, reached by
    administrators through IAP TCP forwarding.
- **Data tier (data subnet):** a 3-node **Adabas cluster** with one VM per zone
  (primary in `-a`, secondaries in `-b` and `-c`). Each VM runs Adabas with AEL and the
  Adabas REST Server on its own Hyperdisk Balanced volumes (CMEK) for ASSO/DATA/WORK.
  The primary handles reads and writes over **ADATCP/S** and synchronizes to the
  secondaries, which serve reads.
- **Natural Batch:** runs on each Adabas VM. The VM in zone `-a` is the **primary** and
  runs batch jobs against the Adabas primary; the VMs in zones `-b` and `-c` are
  **failover** instances that take over if the primary or its zone is lost.

## Key New Products

### Natural OpenAPI Server
The Natural OpenAPI Server turns existing Natural subprograms into REST APIs without
changes to the Natural code.

- **API-first access:** each exposed subprogram becomes a REST endpoint with an
  OpenAPI specification that client teams can use to generate code and test.
- **Active-active instances:** each API call is handled independently, so every VM
  in the backend service can serve traffic at the same time.
- **Zone resilience:** VMs in three zones keep the API available if a zone is lost.
- **Secure by default:** HTTPS for transport and OIDC for authentication, integrated
  with Cloud Identity, Workforce Identity Federation, or another OIDC provider.
- **Integration ready:** can sit behind an API management gateway of the customer's
  choice for rate limiting, versioning, and a developer portal.

### Natural Subprogram Server
The Natural Subprogram Server is the runtime that runs Natural subprograms requested
by remote callers.

- **Redundant execution:** VMs in two zones behind the internal passthrough Network
  Load Balancer, so the API layer always has a healthy target in another zone.
- **Isolation from online users:** API-driven calls run in their own server
  processes, separate from interactive Natural sessions.
- **Elastic capacity:** add VMs or server processes to scale API throughput without
  changing the application.
- **Consistent security:** Natural Security rules apply to subprograms run through the
  server, just as they do for online users.

### Why It Matters
With the Natural OpenAPI Server and the Natural Subprogram Server deployed across
zones, Natural business logic can be offered as highly available APIs alongside the
existing online applications. The same resilient application and data layers serve
both channels.

## Cloud & Security Resources
- **Compute Engine (multi-zone)** – web, application, and data VMs across three zones,
  web and app tiers in regional managed instance groups.
- **Internal Application Load Balancer + Cloud Armor** – HTTPS entry point in the host
  project; TLS termination and a regional security policy in front of the Natural
  OpenAPI Server and Natural Availability Server VMs. Requires a proxy-only subnet in
  the region.
- **Internal passthrough Network Load Balancer** – distributes calls from the web tier
  to healthy Natural Subprogram Server and Natural Web/IO VMs.
- **Cloud Interconnect / HA VPN + Cloud Router** – connects the customer network to
  the Shared VPC.
- **Cloud NGFW & Firewall Rules** – tier-scoped rules; HTTPS to the web tier only from
  the proxy-only subnet and health-check ranges, Natural ports only from the web tier,
  Adabas ports only from the Natural VMs.
- **Identity-Aware Proxy (IAP) TCP forwarding** – administrative access to Adabas
  Manager and the workload VMs without bastion hosts or external IPs.
- **Memorystore (Redis / Valkey)** – shared session state for the Natural Availability
  Server VMs.
- **Hyperdisk Balanced / Persistent Disk** – encrypted persistent storage for each VM.
- **Cloud DNS private zones, Private Service Connect & Cloud NAT** – private name
  resolution, private access to Google APIs, and outbound updates without external IPs.
- **Cloud Identity (OIDC)** – identity provider for API authentication.
- **Secret Manager & Certificate Manager** – TLS certificates and secrets for the API
  layer.
- **Service Accounts (IAM)** – least-privilege access to Google Cloud services
  (backups, monitoring, secrets).
- **Cloud KMS** – customer-managed keys for disks, Cloud Storage, and secrets.
- **Backup and DR, Cloud Storage** – disk snapshots and backup storage.
- **Cloud Logging, Cloud Monitoring, VPC Flow Logs, Security Command Center** –
  metrics, logs, alerts, network logging, and threat detection.
- **Encryption** – encryption in transit and at rest across the API, application,
  and data layers.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on GCP - Resilient HA-v3.0.0.gif`
- `Adabas and Natural on GCP - Resilient HA-v3.0.0.drawio` (editable source)

## Intended Use
Production workloads requiring high availability, business continuity, disaster
recovery, and highly available API access to Natural business logic. For simpler
development setups use the Basic blueprint; for container-native scaling use the
Advanced blueprint.
