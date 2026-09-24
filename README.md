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

---

## 📘 About Adabas & Natural

Adabas is a high-performance, enterprise-grade database, and Natural is its
application development and runtime environment. Together they power core business
systems in banking, insurance, government, healthcare, and manufacturing. These
blueprints give those organizations a consistent, repeatable path to run the same
trusted workloads on cloud infrastructure.

---

## 📂 Repository Structure

```
cloud-blueprints/
├── aws/
│   └── adabas-natural/
│       ├── basic/         └── v2.0.0/   (README + docs, parameters, scripts, templates)
│       ├── resilientHA/   └── v2.0.0/   (README + docs, parameters, scripts, templates)
│       └── advance/       └── v2.0.0/   (README + docs, manifests, parameters, scripts, templates)
├── azure/
│   └── adabas-natural/
│       ├── basic/         └── v2.0.0/
│       ├── resilientHA/   └── v2.0.0/
│       └── advance/       └── v2.0.0/
└── gcp/
    └── adabas-natural/
        ├── basic/         └── v2.0.0/
        ├── resilientHA/   └── v2.0.0/
        └── advance/       └── v2.0.0/
```

Every blueprint is versioned (e.g. `v2.0.0`) and contains:

| Folder        | Contents                                                             |
| ------------- | ------------------------------------------------------------------- |
| `README.md`   | Architecture overview: layers, cloud resources, and security model. |
| `docs/`       | Annotated architecture diagrams for each variant.                   |
| `manifests/`  | Placeholder for Kubernetes manifests (Advanced blueprint only).     |
| `parameters/` | Placeholder for environment-specific configuration values.          |
| `scripts/`    | Placeholder for provisioning/automation assets.                     |
| `templates/`  | Placeholder for Infrastructure-as-Code templates.                   |

---

## 🏗️ Blueprint Catalog

Three blueprint models are available for each cloud provider, forming a maturity path
from a simple lift-and-shift to a fully cloud-native deployment.

| Blueprint        | Model                     | Availability           | Typical Use Case                       |
| ---------------- | ------------------------- | ---------------------- | -------------------------------------- |
| **Basic**        | VM / IaaS                 | Single instance        | Dev/Test, prototyping, lift-and-shift  |
| **Resilient HA** | VM / IaaS, load balanced  | Multi-AZ / Multi-zone  | Production, business continuity        |
| **Advanced**     | Containers / Kubernetes   | Multi-zone, autoscaled | Cloud-native, elastic scaling          |

All blueprints share the same core principle: a clear separation between the
**application layer (Natural)** and the **data layer (Adabas)**, with network
isolation and least-privilege access between them.

### Basic
VM-based deployment for getting started quickly. Variants:
- **Development – Single Instance:** Adabas and Natural co-located on one VM.
- **Distributed:** Adabas and Natural on separate VMs for independent sizing and
  security.
- **Production – Single Instance** *(AWS & Azure):* a single hardened VM for small
  production footprints.

### Resilient HA
Production-grade, highly available deployment. Application nodes are spread across
Availability Zones behind a load balancer; database nodes run in separate zones with
replication. Includes a security & encryption-focused variant covering in-transit and
at-rest protection.

### Advanced
Cloud-native deployment on managed Kubernetes (**Amazon EKS**, **Azure AKS**, or
**Google GKE**). Natural runs as a stateless Deployment for horizontal scaling; Adabas
runs as a stateful StatefulSet with persistent volumes for reliable database
operation.

---

## ☁️ Cloud Provider Coverage

The same architecture is expressed with each provider's native services:

| Capability            | AWS                     | Azure                        | GCP                          |
| --------------------- | ----------------------- | ---------------------------- | ---------------------------- |
| Compute               | EC2                     | Virtual Machines             | Compute Engine               |
| Block storage         | EBS                     | Managed Disks                | Persistent Disks             |
| Network isolation     | VPC / Subnets           | VNet / Subnets               | VPC / Subnets                |
| Firewalling           | Security Groups         | Network Security Groups      | Firewall Rules               |
| Load balancing        | Elastic Load Balancer   | Azure Load Balancer          | Cloud Load Balancing         |
| Managed Kubernetes    | EKS                     | AKS                          | GKE                          |
| Object storage        | S3                      | Blob Storage                 | Cloud Storage                |
| Workload identity     | IAM Roles / IRSA        | Managed Identity             | IAM Service Accounts         |

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
- **Migration & Modernization Teams** – planning lift-and-shift or cloud-native
  transitions with a consistent, repeatable reference.
- **Security & Compliance Officers** – reviewing network segmentation, least-privilege
  access, and encryption posture.

Typical adopters are regulated, large-scale organizations in **banking, insurance,
government, healthcare, and manufacturing** running core systems on Adabas & Natural.

---

## 🎯 Why Use These Blueprints

- **Standardize** Adabas & Natural deployments across AWS, Azure, and GCP.
- **Accelerate** lift-and-shift and modernization initiatives with a proven starting
  point.
- **Reduce risk** by applying cloud best practices for security, high availability,
  and network segmentation.
- **Choose the right model** for each workload with a clear Basic → Resilient HA →
  Advanced maturity path.

---

## 🚀 Getting Started

1. **Pick your cloud provider** – navigate to `aws/`, `azure/`, or `gcp/`.
2. **Choose a blueprint** – `basic/`, `resilientHA/`, or `advance/` based on your
   availability and scaling needs.
3. **Open the versioned README** – e.g. `aws/adabas-natural/resilientHA/v2.0.0/README.md`
   for the architecture overview and layer/security guidance.
4. **Review the diagrams** – see the `docs/` folder for annotated architecture
   diagrams of each variant.
5. **Provision and install** – build the described cloud resources and install the
   Adabas and Natural components following your organization's standards.

---

## 🔖 Versioning

Blueprints are versioned per provider and model (current: **v2.0.0**). Version folders
let you evolve architectures over time without breaking references to earlier
releases.

---

## 📄 License

This repository is distributed under the terms of the [LICENSE](LICENSE) file included
in this repository.
