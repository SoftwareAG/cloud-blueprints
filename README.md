# Cloud Blueprints for Adabas & Natural

**Reference architectures for running Adabas & Natural workloads on Amazon Web Services (AWS), Microsoft Azure and Google Cloud (GCP).**

![Release](https://img.shields.io/badge/release-v3.0.0-blue)
![AWS](https://img.shields.io/badge/AWS-Basic%20%7C%20Resilient%20HA%20%7C%20EKS-FF9900?logo=amazonwebservices&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-Basic%20%7C%20Resilient%20HA%20%7C%20AKS-0078D4?logo=microsoftazure&logoColor=white)
![Google Cloud](https://img.shields.io/badge/Google%20Cloud-Basic%20%7C%20Resilient%20HA%20%7C%20GKE-4285F4?logo=googlecloud&logoColor=white)

Each blueprint gives you an annotated architecture diagram and layer-by-layer guidance for a target architecture. Use it to plan, standardize and de-risk moving business-critical Adabas & Natural applications to the cloud. The blueprints describe **what** to build and **why**. You provision the cloud resources and install the Adabas (data) and Natural (application) components on them according to your organization's standards.

---

## Contents

- [About Adabas & Natural](#-about-adabas--natural)
- [Use cases](#-use-cases)
- [Blueprint catalog](#️-blueprint-catalog)
- [Blueprint models](#-blueprint-models)
- [Common architecture principles](#-common-architecture-principles)
- [Cloud provider coverage](#️-cloud-provider-coverage)
- [Who should use these blueprints](#-who-should-use-these-blueprints)
- [Getting started](#-getting-started)
- [Repository structure](#-repository-structure)
- [Versioning](#-versioning)
- [Feedback](#-feedback)
- [Disclaimer](#️-disclaimer)
- [License](#-license)

---

## 📘 About Adabas & Natural

**Adabas** is a high-performance, enterprise-grade database. **Natural** is its application development and runtime environment. Together they run core business systems in banking, insurance, government, healthcare and manufacturing, often for decades and at very high transaction volumes. These blueprints give those organizations a consistent, repeatable way to run the same trusted workloads on cloud infrastructure, from a single development VM up to a three-zone production cluster.

---

## 🎯 Use cases

| Use case | What the blueprints provide | Start with |
|---|---|---|
| **Mainframe / data center exit** | A proven target architecture on IaaS for rehosted A&N applications | Basic → Resilient HA |
| **Production with high availability** | Zone-redundant web, app and data tiers with a 3-node Adabas cluster | Resilient HA |
| **API enablement of Natural logic** | REST/OpenAPI access to existing Natural subprograms, with no code changes | Any v3.0.0 blueprint |
| **Browser access to Natural online applications** | Natural Availability Server with shared session state | Resilient HA, Advanced |
| **Container platform adoption** | The same A&N architecture as workloads on managed Kubernetes | Advanced |
| **Dev, test and QA stages** | Low-cost, private single-VM or two-VM environments | Basic |

---

## 🗂️ Blueprint catalog

Current release **v3.0.0**. Each link opens the blueprint's README. The `docs/` folder next to it holds the architecture diagrams.

| Blueprint | AWS | Azure | Google Cloud |
|---|---|---|---|
| **Basic** (VM / IaaS) | [aws/…/basic/v3.0.0](aws/adabas-natural/basic/v3.0.0/README.md) | [azure/…/basic/v3.0.0](azure/adabas-natural/basic/v3.0.0/README.md) | [gcp/…/basic/v3.0.0](gcp/adabas-natural/basic/v3.0.0/README.md) |
| **Resilient HA** (VM / IaaS, multi-zone) | [aws/…/resilientHA/v3.0.0](aws/adabas-natural/resilientHA/v3.0.0/README.md) | [azure/…/resilientHA/v3.0.0](azure/adabas-natural/resilientHA/v3.0.0/README.md) | [gcp/…/resilientHA/v3.0.0](gcp/adabas-natural/resilientHA/v3.0.0/README.md) |
| **Advanced** (managed Kubernetes) | [aws/…/advance/v3.0.0](aws/adabas-natural/advance/v3.0.0/README.md) (EKS) | [azure/…/advance/v3.0.0](azure/adabas-natural/advance/v3.0.0/README.md) (AKS) | [gcp/…/advance/v3.0.0](gcp/adabas-natural/advance/v3.0.0/README.md) (GKE) |

The previous release, **v2.0.0**, is still available in the `v2.0.0/` folder of each blueprint.

---

## 🏗️ Blueprint models

The three models form a maturity path from a single VM to a zone-redundant deployment on a container platform. Choose the model per workload and stage. A customer often runs Basic for development and test, and Resilient HA or Advanced for production.

| Blueprint | Model | Availability | Typical use case |
|---|---|---|---|
| **Basic** | VM / IaaS | Single instance per layer | Development, prototyping, API pilots, test stages, lift-and-shift without HA |
| **Resilient HA** | VM / IaaS, load balanced | Multiple zones; 3-node Adabas cluster | Production, business continuity, highly available APIs |
| **Advanced** | Containers on managed Kubernetes (EKS, AKS, GKE) | 3 zones, horizontal scaling | Production on a standardized container platform, elastic scaling |

### Basic

VM-based deployment for getting started quickly. Every layer, including the Natural OpenAPI Server and Natural Subprogram Server, runs on one or two VMs in private subnets.

- **Development, single instance.** All layers on one VM, together with the Natural Development Server (NDV) for NaturalONE. There is no public ingress: developers and testers reach it privately over Direct Connect, ExpressRoute, Interconnect or VPN.
- **Distributed.** Adabas and Natural on separate VMs, each in its own subnet, so each can be sized and secured on its own. Suited to INT, UAT and QA, or to production where HA is not required. The VMs sit behind the provider's L7 ingress with WAF.
- **Production, single instance** *(GCP)*. A single hardened VM for small production footprints. It is described in the GCP README and has no separate diagram.

### Resilient HA

Production-grade, highly available deployment in three tiers of private subnets:

- **Web tier.** Natural OpenAPI Server and Natural Availability Server instances in several zones (two on AWS and Azure, three on GCP), behind the provider's L7 load balancer with WAF. The Natural Availability Server keeps session state in a managed Redis/Valkey cache, so any instance can serve any user.
- **App tier.** Natural Subprogram Server and Natural Web I/O Server instances, with Natural Security, in two zones behind an internal load balancer.
- **Data tier.** A 3-node, shared-nothing Adabas cluster with one primary and two secondaries, one per zone. Each node has its own encrypted volumes, Adabas Encryption for Linux (AEL) and the Adabas REST Server. Natural Batch is installed on every Adabas node and active on one node at a time, and fails over with the cluster.
- **Management.** An Adabas Manager instance for administration, reached through Session Manager, Azure Bastion or IAP. It is not on the request path.

### Advanced

The Resilient HA architecture on managed Kubernetes (**Amazon EKS**, **Azure AKS** or **Google GKE**). The VMs are replaced by Kubernetes workloads that run the official Adabas and Natural container images:

- **Ingress.** AWS Load Balancer Controller (ALB with AWS WAF), Application Gateway Ingress Controller (Application Gateway with WAF) or GKE Gateway (internal Application Load Balancer with Cloud Armor). Each provides one endpoint for the Natural OpenAPI Server and one for the Natural Availability Server.
- **`adanat-web` namespace.** Natural OpenAPI Server and Natural Availability Server Deployments. This is the only namespace exposed through the ingress.
- **`adanat` namespace.** Natural Subprogram Server, Natural Web I/O Server and Adabas Manager pods, plus a 3-replica Adabas **StatefulSet** with one pod per zone on zonal persistent volumes. Natural Batch runs as a single pod, a one-replica Deployment or a CronJob, against the Adabas primary. Kubernetes reschedules it if it fails.
- **Platform controls.** Private cluster API endpoint, network policies between tiers, topology spread across zones, workload identity for pod access to cloud services, and secrets mounted through the Secrets Store CSI driver.

> The Advanced blueprints describe the target platform and workload layout using standard Kubernetes objects: Deployments, StatefulSets, Services and network policies. They do not depend on a Kubernetes operator.

---

## 🧭 Common architecture principles

All v3.0.0 cloud blueprints share these principles:

- **Separated layers.** An **API/web layer** (Natural OpenAPI Server, Natural Availability Server), an **application layer** (Natural, Natural Subprogram Server, Natural Web I/O Server, Natural Security) and a **data layer** (Adabas). The layers are isolated at the network level, with least-privilege access between them. Natural reaches Adabas over **ADATCP/S**.
- **Hub-and-spoke network.** Shared ingress, inspection, DNS and private service endpoints sit in a hub: an AWS hub VPC on Transit Gateway, an Azure hub VNet, or a GCP Shared VPC host project. The Adabas & Natural workload sits in a spoke.
- **No public IPs on workloads.** Workloads run in private subnets. Admin access goes through Session Manager, Azure Bastion or IAP, with no open SSH ports and no jump hosts. Internal and admin traffic arrives privately over Direct Connect, ExpressRoute, Cloud Interconnect or VPN.
- **Controlled ingress.**
  - On **AWS and Azure**, the only public HTTPS entry point (where a blueprint offers one) is an L7 load balancer with WAF in the hub.
  - On **GCP**, the load balancer is internal and clients connect only from the customer network.
  - The Basic Development variant has no ingress at all.
- **Encrypted end to end.** Clients use HTTPS with OIDC. Traffic between tiers uses TLS. Disks, backups and secrets are encrypted with customer-managed keys. Adabas data is encrypted at rest with AEL.
- **Zone resilience** *(Resilient HA, Advanced).* Every tier survives the loss of an instance or a whole zone. Regional DR (AWS Elastic Disaster Recovery, Azure Site Recovery, or cross-region backup) is described as an optional extension.

---

## ☁️ Cloud provider coverage

Each provider's native services implement the same architecture:

| Capability | AWS | Azure | GCP |
|---|---|---|---|
| Compute | EC2 | Virtual Machines | Compute Engine |
| Block storage | EBS (gp3 / io2) | Managed Disks (Premium SSD) | Hyperdisk Balanced / Persistent Disk |
| Hub-and-spoke network | Hub & workload VPCs, Transit Gateway | Hub & spoke VNets (peering) | Shared VPC (host & service projects) |
| Private connectivity | Direct Connect / Site-to-Site VPN | ExpressRoute / VPN Gateway | Cloud Interconnect / HA VPN + Cloud Router |
| L7 ingress & WAF | Application Load Balancer + AWS WAF | Application Gateway (WAF_v2) | Internal Application LB + Cloud Armor |
| Internal load balancing | Network Load Balancer | Azure Load Balancer (internal) | Internal passthrough Network LB |
| Network inspection | AWS Network Firewall, Security Groups | Azure Firewall, NSGs | Cloud NGFW, firewall rules |
| Admin access | Systems Manager Session Manager | Azure Bastion | IAP TCP forwarding |
| Session cache | ElastiCache (Redis OSS / Valkey) | Azure Cache for Redis | Memorystore (Redis / Valkey) |
| API management (optional) | Amazon API Gateway | Azure API Management | Customer's choice of API gateway |
| Managed Kubernetes | EKS | AKS | GKE |
| Kubernetes ingress | AWS Load Balancer Controller | Application Gateway Ingress Controller | GKE Gateway |
| Container registry | ECR | Azure Container Registry | Artifact Registry |
| Identity (OIDC) | IAM Identity Center / Cognito | Microsoft Entra ID | Cloud Identity / Workforce Identity Federation |
| Workload identity | IAM roles / EKS Pod Identity | Managed Identity / Workload Identity | Service accounts / Workload Identity Federation |
| Secrets & certificates | Secrets Manager, Certificate Manager | Key Vault | Secret Manager, Certificate Manager |
| Encryption keys | AWS KMS | Disk Encryption Sets / Key Vault | Cloud KMS (CMEK) |
| Backup | AWS Backup, S3 | Backup Vault (AKS Backup), Blob Storage | Backup and DR / Backup for GKE, Cloud Storage |
| Monitoring & threat detection | CloudWatch, GuardDuty | Azure Monitor, Defender for Containers | Cloud Monitoring/Logging, Security Command Center |
| Regional DR (optional) | AWS Elastic Disaster Recovery | Azure Site Recovery | Cross-region backup / replication |

---

## 👥 Who should use these blueprints

| Role | How the blueprints help |
|---|---|
| **Enterprise & cloud architects** | Select a target architecture and validate it against security, availability and compliance requirements |
| **Platform & infrastructure engineers** | Provision the network, compute, storage, Kubernetes and security resources for each layer |
| **Adabas & Natural administrators** | Install and operate the database and application components on the provisioned infrastructure |
| **Integration & API teams** | Expose Natural business logic as REST APIs through the Natural OpenAPI Server |
| **Migration & modernization teams** | Plan rehosting and cloud transitions with a consistent, repeatable reference |
| **Security & compliance officers** | Review network segmentation, least-privilege access, identity and encryption |

Typical adopters are large, regulated organizations in **banking, insurance, government, healthcare and manufacturing** that run core systems on Adabas & Natural.

---

## 🚀 Getting started

1. **Pick a cloud provider.** Choose `aws/`, `azure/` or `gcp/`.
2. **Choose a model.** Pick `basic/`, `resilientHA/` or `advance/` based on your availability and scaling needs. Use the [use cases](#-use-cases) table as a guide.
3. **Read the versioned README.** For example, [azure/adabas-natural/resilientHA/v3.0.0](azure/adabas-natural/resilientHA/v3.0.0/README.md) covers the network layout, tiers, cloud resources and security model.
4. **Review the diagrams.** The `docs/` folder holds the architecture diagram for each variant.
5. **Provision and install.** Build the cloud resources with your organization's IaC tooling (for example Terraform, CloudFormation or Bicep). Then install the Adabas and Natural components, or deploy the container images for Advanced, following the product documentation.

---

## 📂 Repository structure

```text
cloud-blueprints/
├── aws/
│   └── adabas-natural/
│       ├── basic/          v2.0.0/  v3.0.0/
│       ├── resilientHA/    v2.0.0/  v3.0.0/
│       └── advance/        v2.0.0/  v3.0.0/    (EKS)
├── azure/
│   └── adabas-natural/
│       ├── basic/          v2.0.0/  v3.0.0/
│       ├── resilientHA/    v2.0.0/  v3.0.0/
│       └── advance/        v2.0.0/  v3.0.0/    (AKS)
└── gcp/
    └── adabas-natural/
        ├── basic/          v2.0.0/  v3.0.0/
        ├── resilientHA/    v2.0.0/  v3.0.0/
        └── advance/        v2.0.0/  v3.0.0/    (GKE)
```

Each versioned blueprint folder contains:

| Item | Contents |
|---|---|
| `README.md` | Architecture overview: network layout, tiers, key products, cloud resources, security model, intended use |
| `docs/` | Architecture diagrams (`.gif`) for each variant |

---

## 🔖 Versioning

Blueprints are versioned per provider and model. Earlier releases stay in their version folders, so existing references keep working while the architectures evolve.

| Version | Scope | Highlights |
|---|---|---|
| **v3.0.0** | AWS, Azure, GCP | Adds the **Natural OpenAPI Server** and **Natural Subprogram Server** to every blueprint. Existing Natural subprograms become REST endpoints described by OpenAPI, with no changes to the Natural code, and API calls run in their own server processes under Natural Security. |
| **v2.0.0** | AWS, Azure, GCP | Basic, Resilient HA and Advanced blueprints for all three providers, including the Natural Availability Server |

---

## 💬 Feedback

Questions, corrections and suggestions for new blueprints are welcome. Please open a GitHub issue that names the provider, model and version (for example `azure / resilientHA / v3.0.0`).

---

## ⚠️ Disclaimer

These blueprints are reference architectures provided for guidance. They are a starting point for your own design, not a certified production configuration. Before you deploy to production, validate sizing, security, compliance, licensing and product support levels against your organization's requirements and the current Adabas & Natural product documentation. Cloud service names and capabilities change over time, so check each provider's current documentation.

---

## 📄 License

This repository is distributed under the terms of the [LICENSE](LICENSE) file included in this repository.
