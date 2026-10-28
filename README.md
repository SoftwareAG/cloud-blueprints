# Cloud Blueprints for Adabas & Natural

Enterprise reference architectures for deploying **Adabas & Natural** workloads on
the major cloud hyperscalers — **Amazon Web Services (AWS)**, **Microsoft Azure**,
and **Google Cloud Platform (GCP)**.

Each blueprint provides a vendor-validated architecture, annotated diagrams, and
layer-by-layer guidance to help you plan, standardize, and de-risk the move of
mission-critical Adabas & Natural applications to the cloud. The blueprints describe
*what* to build and *why*; you provision the cloud resources and install the Adabas
(data) and Natural (application) components on them according to your organization's
standards.

The current release is **v3.0.0**. It adds API access to Natural business logic
through the **Natural OpenAPI Server** and **Natural Subprogram Server**, and places
every blueprint on a private hub-and-spoke network with no internet exposure.

---

## 📘 About Adabas & Natural

Adabas is a high-performance, enterprise-grade database, and Natural is its
application development and runtime environment. Together they power core business
systems in banking, insurance, government, healthcare, and manufacturing. These
blueprints give those organizations a consistent, repeatable path to run the same
trusted workloads on cloud infrastructure.

---

## 📂 Repository Structure

```text
cloud-blueprints/
├── aws/
│   └── adabas-natural/
│       ├── basic/         ├── v2.0.0/
│       │                  └── v3.0.0/   (README + docs)
│       ├── resilientHA/   ├── v2.0.0/
│       │                  └── v3.0.0/
│       └── advance/       ├── v2.0.0/
│                          └── v3.0.0/
├── azure/
│   └── adabas-natural/
│       ├── basic/         ├── v2.0.0/
│       │                  └── v3.0.0/
│       ├── resilientHA/   ├── v2.0.0/
│       │                  └── v3.0.0/
│       └── advance/       ├── v2.0.0/
│                          └── v3.0.0/
└── gcp/
    └── adabas-natural/
        ├── basic/         ├── v2.0.0/
        │                  └── v3.0.0/
        ├── resilientHA/   ├── v2.0.0/
        │                  └── v3.0.0/
        └── advance/       ├── v2.0.0/
                           └── v3.0.0/
```

Every blueprint is versioned (e.g. `v3.0.0`) and contains:

| Item        | Contents                                                                         |
| ----------- | -------------------------------------------------------------------------------- |
| `README.md` | Architecture overview: network layout, layers, cloud resources, security model. |
| `docs/`     | Architecture diagrams (`.gif`) for each variant.                                 |

---

## 🏗️ Blueprint Catalog

Three blueprint models are available for each cloud provider, forming a maturity path
from a simple lift-and-shift to a fully cloud-native deployment.

| Blueprint        | Model                     | Availability                  | Typical Use Case                                   |
| ---------------- | ------------------------- | ----------------------------- | -------------------------------------------------- |
| **Basic**        | VM / IaaS                 | Single instance               | Dev/Test, prototyping, API pilots, lift-and-shift  |
| **Resilient HA** | VM / IaaS, load balanced  | 3 Availability Zones          | Production, business continuity, HA APIs           |
| **Advanced**     | Containers / Kubernetes   | 3 Availability Zones, autoscaled | Cloud-native, elastic scaling                   |

All blueprints share the same core principles:

- **Separated layers** – an **API layer** (Natural OpenAPI Server), an **application
  layer** (Natural, Natural Subprogram Server), and a **data layer** (Adabas), with
  network isolation and least-privilege access between them. Natural reaches Adabas
  over **ADATCP/S**.
- **Private by default** – client traffic arrives from the customer network over
  private connectivity (Direct Connect / ExpressRoute / Cloud Interconnect, or VPN).
  Load balancers are internal, and workloads have no public IPs.
- **Hub-and-spoke network** – shared ingress, inspection, DNS, and private service
  endpoints sit in a hub (AWS hub VPC on Transit Gateway, Azure hub VNet, GCP Shared
  VPC host project), and the Adabas & Natural workload sits in a spoke.
- **Encrypted end to end** – HTTPS with OIDC for API callers, TLS between tiers, and
  customer-managed keys for disks, backups, and secrets.

### What's new in v3.0.0

- **Natural OpenAPI Server** – publishes existing Natural subprograms as
  OpenAPI-described REST endpoints over HTTPS, with no changes to the Natural code.
- **Natural Subprogram Server** – runs the subprograms behind each API call in its own
  server processes, separate from interactive Natural sessions, with Natural Security
  applied.
- **Natural Availability Server** *(Resilient HA and Advanced)* – multiple instances
  sharing session state in a managed Redis/Valkey cache, so any instance can serve any
  user.
- **Separate endpoints per product** – the ingress publishes one host name for the
  Natural OpenAPI Server and one for the Natural Availability Server, each with its own
  TLS certificate, WAF rules, and health check.
- **3-zone Adabas cluster** – a primary and two secondaries, one per zone, each with
  its own encrypted volumes, with Adabas Encryption for Linux (AEL) and the Adabas REST
  Server.

### Basic

VM-based deployment for getting started quickly. All layers, including the Natural
OpenAPI Server and Natural Subprogram Server, run on one or two VMs in private subnets
behind an internal load balancer with a WAF. Variants:
- **Development – Single Instance:** all layers co-located on one VM.
- **Distributed:** Adabas and Natural on separate VMs, each in its own subnet, for
  independent sizing and security.
- **Production – Single Instance:** a single hardened VM for small production
  footprints, with the API published only over HTTPS. *(Described in the README; no
  separate diagram.)*

### Resilient HA

Production-grade, highly available deployment across three Availability Zones, in
three tiers of private subnets:
- **Web tier:** Natural OpenAPI Server and Natural Availability Server instances in
  every zone, behind an internal Application Load Balancer / Application Gateway with
  WAF.
- **App tier:** Natural Subprogram Server and Natural Web I/O instances (with Natural
  Security) in two zones behind an internal load balancer, plus an Adabas Manager
  instance for administration through Session Manager / Azure Bastion / IAP.
- **Data tier:** a 3-node Adabas cluster, one instance per zone. Natural Batch runs on
  each Adabas instance, with a primary in the first zone and failover instances in the
  other two.

### Advanced

The Resilient HA architecture on managed Kubernetes (**Amazon EKS**, **Azure AKS**, or
**Google GKE**), with the VMs replaced by Kubernetes workloads:
- **Ingress** – AWS Load Balancer Controller (internal ALB + WAF), Application Gateway
  Ingress Controller (Application Gateway + WAF), or GKE Gateway (internal Application
  Load Balancer + Cloud Armor).
- **`adanat-web` namespace** – Natural OpenAPI Server and Natural Availability Server
  Deployments, the only namespace exposed through the ingress.
- **`adanat` namespace** – Natural Subprogram Server, Natural Web I/O, and Adabas
  Manager pods; a 3-replica Adabas **StatefulSet** with zonal persistent volumes; and
  Natural Batch in separate pods co-located with each Adabas pod (primary and failover).
- Private cluster API endpoint, network policies between tiers, workload identity for
  pod access to cloud services, and secrets mounted through the Secrets Store CSI driver.

---

## ☁️ Cloud Provider Coverage

The same architecture is expressed with each provider's native services:

| Capability              | AWS                                    | Azure                              | GCP                                          |
| ----------------------- | -------------------------------------- | ---------------------------------- | -------------------------------------------- |
| Compute                 | EC2                                    | Virtual Machines                   | Compute Engine                               |
| Block storage           | EBS (gp3 / io2)                        | Managed Disks (Premium SSD)        | Hyperdisk Balanced / Persistent Disk         |
| Hub-and-spoke network   | Hub & workload VPCs, Transit Gateway   | Hub & spoke VNets (peering)        | Shared VPC (host & service projects)         |
| Private connectivity    | Direct Connect / Site-to-Site VPN      | ExpressRoute / VPN Gateway         | Cloud Interconnect / HA VPN + Cloud Router   |
| Ingress & WAF           | Internal ALB + AWS WAF                 | Application Gateway + WAF          | Internal Application LB + Cloud Armor        |
| Internal load balancing | Network Load Balancer                  | Azure Load Balancer (internal)     | Internal passthrough Network LB              |
| Network inspection      | AWS Network Firewall, Security Groups  | Azure Firewall, NSGs               | Cloud NGFW, Firewall Rules                   |
| Admin access            | Systems Manager Session Manager        | Azure Bastion                      | IAP TCP forwarding                           |
| Session cache           | ElastiCache (Redis OSS / Valkey)       | Azure Cache for Redis              | Memorystore (Redis / Valkey)                 |
| API management (opt.)   | Amazon API Gateway                     | Azure API Management               | Customer's choice of API gateway             |
| Managed Kubernetes      | EKS                                    | AKS                                | GKE                                          |
| Container registry      | ECR                                    | Azure Container Registry           | Artifact Registry                            |
| Identity (OIDC)         | IAM Identity Center / Cognito          | Microsoft Entra ID                 | Cloud Identity / Workforce Identity Fed.     |
| Workload identity       | IAM Roles / EKS Pod Identity           | Managed Identity / Workload Identity | Service Accounts / Workload Identity Fed.  |
| Secrets & certificates  | Secrets Manager, Certificate Manager   | Key Vault                          | Secret Manager, Certificate Manager          |
| Encryption keys         | AWS KMS                                | Disk Encryption Sets / Key Vault   | Cloud KMS (CMEK)                             |
| Backup                  | AWS Backup, S3                         | Backup Vault (AKS Backup), Blob Storage | Backup and DR / Backup for GKE, Cloud Storage |
| Monitoring & threat detection | CloudWatch, GuardDuty            | Azure Monitor, Defender for Containers | Cloud Monitoring/Logging, Security Command Center |

---

## 👥 Who Should Use These Blueprints

These blueprints are designed for enterprise teams modernizing Adabas & Natural
estates:

- **Enterprise & Cloud Architects** – selecting a target architecture and validating
  it against security, availability, and compliance requirements.
- **Platform & Infrastructure Engineers** – provisioning the network, compute,
  storage, and security resources for each layer.
- **Adabas & Natural Administrators** – installing and operating the database and
  application components on the provisioned infrastructure.
- **Integration & API Teams** – exposing Natural business logic as REST APIs through
  the Natural OpenAPI Server.
- **Migration & Modernization Teams** – planning lift-and-shift or cloud-native
  transitions with a consistent, repeatable reference.
- **Security & Compliance Officers** – reviewing network segmentation, least-privilege
  access, and encryption posture.

Typical adopters are regulated, large-scale organizations in **banking, insurance,
government, healthcare, and manufacturing** running core systems on Adabas & Natural.

---

## 🎯 Why Use These Blueprints

- **Standardize** Adabas & Natural deployments across AWS, Azure, and GCP.
- **Open up Natural business logic** as secure, highly available REST APIs without
  rewriting it.
- **Accelerate** lift-and-shift and modernization initiatives with a proven starting
  point.
- **Reduce risk** by applying cloud best practices for security, high availability,
  private connectivity, and network segmentation.
- **Choose the right model** for each workload with a clear Basic → Resilient HA →
  Advanced maturity path.

---

## 🚀 Getting Started

1. **Pick your cloud provider** – navigate to `aws/`, `azure/`, or `gcp/`.
2. **Choose a blueprint** – `basic/`, `resilientHA/`, or `advance/` based on your
   availability and scaling needs.
3. **Open the versioned README** – e.g. `azure/adabas-natural/resilientHA/v3.0.0/README.md`
   for the network layout, layers, and security guidance.
4. **Review the diagrams** – see the `docs/` folder for the architecture
   diagrams of each variant.
5. **Provision and install** – build the described cloud resources and install the
   Adabas and Natural components following your organization's standards.

---

## 🔖 Versioning

Blueprints are versioned per provider and model. Version folders let you evolve
architectures over time without breaking references to earlier releases.

| Version    | Providers       | Highlights                                                                 |
| ---------- | --------------- | -------------------------------------------------------------------------- |
| **v3.0.0** | AWS, Azure, GCP | Adds the **Natural OpenAPI Server**, **Natural Subprogram Server**, and **Natural Availability Server** (shared session cache). Private hub-and-spoke network for every blueprint (AWS hub and workload VPCs on Transit Gateway; Azure hub/spoke VNets; GCP Shared VPC host and service projects) with internal ingress, WAF, and no public IPs. Resilient HA and Advanced span three zones with separate API and online endpoints, a 3-zone Adabas cluster, and primary/failover Natural Batch; Advanced uses the AWS Load Balancer Controller, Application Gateway Ingress Controller, or GKE Gateway for ingress. |
| **v2.0.0** | AWS, Azure, GCP | Basic, Resilient HA, and Advanced blueprints for all three providers. |

---

## 📄 License

This repository is distributed under the terms of the [LICENSE](LICENSE) file included
in this repository.
