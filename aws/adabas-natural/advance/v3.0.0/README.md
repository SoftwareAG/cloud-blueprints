# Adabas & Natural on AWS – Advanced (EKS)

## Overview
The Advanced blueprint runs Adabas & Natural as containers on Amazon Elastic
Kubernetes Service (EKS) across three Availability Zones. It is intended for scalable,
resilient, cloud-native production workloads. It follows the v3.0.0 Resilient HA
reference architecture, with the EC2 instances replaced by Kubernetes workloads.
Customers provision the EKS cluster and supporting resources shown in the diagram and
deploy the official Software AG Adabas and Natural container images.

## What's New in v3.0.0
- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI, deployed as multiple replicas across zones.
- **Natural Subprogram Server** – runs Natural subprograms on behalf of the Natural
  OpenAPI Server, deployed as its own multi-replica Deployment.
- **Natural Availability Server** – multiple replicas sharing session state
  through Amazon ElastiCache.
- **Application Load Balancer as ingress** – an internal Application Load Balancer with
  AWS WAF, managed by the AWS Load Balancer Controller, is the single HTTPS entry
  point, with a separate endpoint for each product: one for the Natural OpenAPI Server
  and one for the Natural Availability Server.
- **3-zone Adabas cluster** – a 3-node Adabas cluster with one pod per Availability
  Zone.
- **Hub-and-spoke network** – a network hub VPC holds the shared inspection and
  administration components and is connected to the EKS workload VPC through AWS
  Transit Gateway.

## Network Layout
- **Customer access:** client traffic arrives from the customer network over AWS
  Direct Connect or Site-to-Site VPN and the Transit Gateway. The Application Load
  Balancer is internal; no Internet Gateway or public IPs are used.
- **Hub VPC:** AWS Network Firewall, VPC interface endpoints shared through Route 53
  private hosted zones, and an SSM-managed admin host (EC2) for cluster
  administration. Transit Gateway, AWS WAF, Route 53, Systems Manager Session Manager,
  and the optional Amazon API Gateway are regional services and are shown with the hub
  they serve.
- **Workload VPC:** dedicated ingress subnets for the internal Application Load
  Balancer – the AWS Load Balancer Controller creates the load balancer in the
  cluster's own VPC – and private subnets in each Availability Zone for the EKS node
  groups and pods (Amazon VPC CNI). Security groups restrict the load balancer, nodes,
  and pods; an S3 gateway endpoint carries backup traffic to Amazon S3.

## Architecture Layers
- **Ingress (workload VPC):** an internal Application Load Balancer with AWS WAF in the
  ingress subnets receives client HTTPS traffic. Through the AWS Load Balancer
  Controller (Kubernetes Ingress, IP targets) it publishes two endpoints, each routed
  to its own Kubernetes Service in the cluster:

  | Load balancer rule (example)     | Kubernetes Service     | Backend pods                  |
  | -------------------------------- | ---------------------- | ----------------------------- |
  | `https://api.adanat.example.com` | `natural-openapi`      | Natural OpenAPI Server        |
  | `https://nas.adanat.example.com` | `natural-availability` | Natural Availability Server   |

  Each endpoint has its own listener rule, TLS certificate (from AWS Certificate
  Manager), WAF rules, and health check, so API traffic and online user traffic can be
  secured, scaled, and monitored independently. Replace the example host names with
  the customer's own DNS names; path-based routing on a single host name (for example
  `/api` and `/nas`) works the same way if separate host names are not wanted.
- **Web tier (`adanat-web` namespace):** Deployments for the **Natural OpenAPI Server**
  and the **Natural Availability Server**, each with at least one replica per zone and
  each behind its own Service. The Natural Availability Server replicas keep session
  state in Amazon ElastiCache, so any replica can serve any user.
- **App tier (`adanat` namespace):** an internal Kubernetes Service (ClusterIP)
  distributes calls from the web tier to:
  - **Natural Subprogram Server** pods (Natural Subprogram Server, Natural Security,
    Natural) in Zones A and B, serving the API calls.
  - **Natural Web/IO** pods (Natural Web/IO, Natural Security, Natural) in Zones A
    and B, serving the online applications.
  - an **Adabas Manager** pod in Zone C for database administration. Administrators
    connect with Session Manager to the admin host in the hub VPC and reach the pod
    through the private EKS API endpoint (`kubectl port-forward`).
- **Data tier (`adanat` namespace):** a 3-replica **StatefulSet** forming the Adabas
  cluster, with one pod per Availability Zone (`adabas-0` primary, `adabas-1` and
  `adabas-2` secondaries). Each pod runs Adabas with AEL and the Adabas REST Server,
  and has its own zonal EBS (gp3 / io2) persistent volume for ASSO/DATA/WORK,
  provisioned by the Amazon EBS CSI driver. The primary handles reads and writes over
  **ADATCP/S** and synchronizes to the secondaries, which serve reads.
- **Natural Batch (`adanat` namespace):** runs in its own pods, separate from the
  Adabas pods but placed next to them — one Natural Batch pod per zone, co-located
  with that zone's Adabas pod (pod affinity). The pod in Zone A is the **primary**
  and runs batch jobs against the Adabas primary; the pods in Zones B and C are
  **failover** instances that take over if the primary batch pod or its zone is lost.
  Keeping batch in separate pods lets it be scaled, scheduled, and restarted without
  touching the database pods.

### Namespaces
- **`adanat-web`** – the web-facing tier: Natural OpenAPI Server and Natural
  Availability Server, with the `natural-openapi` and `natural-availability`
  Services. It is the only namespace exposed through the load balancer ingress.
- **`adanat`** – the Adabas and Natural core: Natural Subprogram Server, Natural
  Web/IO, Natural Batch, Adabas Manager, and the Adabas cluster StatefulSet. It is
  reachable only from `adanat-web` and from administrators.

Keeping the web tier in its own namespace lets it be scaled, updated, and secured
(RBAC, network policies, resource quotas) separately from the core runtime and
database.

Pods are spread across zones with topology spread constraints and anti-affinity, so
the loss of a zone leaves the API, online, and database services running. Because EBS
volumes are zonal, each Adabas pod stays in its own zone and node groups must have
capacity in all three zones.

## Key Components on EKS

### Natural OpenAPI Server
- Turns existing Natural subprograms into OpenAPI-described REST endpoints without
  code changes.
- Exposed through its own Service (`natural-openapi`) and load balancer endpoint
  (e.g. `https://api.adanat.example.com`).
- Runs active-active: each API call is handled independently, so every replica
  behind the Service can serve traffic. Scale with replicas or a Horizontal Pod
  Autoscaler.
- HTTPS at the Application Load Balancer and OIDC authentication with AWS IAM
  Identity Center, Amazon Cognito, or another OIDC provider.

### Natural Subprogram Server
- Runs the Natural subprograms requested through the Natural OpenAPI Server, in its
  own Deployment, separate from interactive Natural workloads.
- Replicas in two zones, reached through an internal Kubernetes Service, so the API
  tier still has a healthy target if a zone is lost. Add a replica in Zone C if the
  API workload needs capacity in all three zones.
- Natural Security rules apply to subprograms run through the server.

### Natural Availability Server
- Web front end for Natural online applications, running as multiple replicas behind
  its own Service (`natural-availability`) and load balancer endpoint
  (e.g. `https://nas.adanat.example.com`).
- Session and application state held in Amazon ElastiCache, so a lost pod or zone
  does not interrupt users.
- Pods can be drained and updated one by one for maintenance without an outage.

### Adabas Cluster (StatefulSet)
- Three pods, one per zone, each with its own persistent volume and independent copy
  of the data — no shared disk.
- Automatic failover to a secondary if the primary pod, node, or zone is lost.
- Rolling maintenance one pod at a time, and read scale-out on the secondaries.
- Data encrypted at rest by Adabas Encryption for Linux (AEL) and EBS encryption with
  AWS KMS customer-managed keys.

## Cloud & Security Resources
- **Amazon Elastic Kubernetes Service (EKS)** – managed control plane and node groups
  across three Availability Zones.
- **Application Load Balancer (internal) + AWS WAF (AWS Load Balancer Controller)** –
  ingress in the workload VPC's ingress subnets, TLS termination, and web application
  firewall, with separate rules for the Natural OpenAPI Server and Natural
  Availability Server endpoints.
- **Hub & workload VPCs with AWS Transit Gateway** – inspection and administration in
  the hub (AWS Network Firewall, VPC interface endpoints, SSM-managed admin host), the
  EKS cluster and its ingress in the workload VPC; Transit Gateway with Site-to-Site
  VPN, Route 53 private hosted zones, Systems Manager Session Manager, and the optional
  Amazon API Gateway as regional services.
- **Private EKS API endpoint** – cluster administration from the admin host only; no
  public API endpoint.
- **Security Groups & S3 gateway endpoint** – security groups for the load balancer,
  nodes, and pods; S3 access from the workload VPC without leaving the AWS network.
- **Kubernetes Services & Network Policies** – internal ClusterIP services between
  tiers; network policies allow only the `adanat-web` pods to reach the Natural
  services in `adanat`, and only the Natural pods to reach the Adabas pods.
- **Persistent Volumes (Amazon EBS gp3 / io2, zonal)** – durable storage for each
  Adabas pod through the EBS CSI driver.
- **Amazon ElastiCache (Redis OSS / Valkey)** – shared session state for the Natural
  Availability Server.
- **Amazon ECR** – Software AG container images.
- **AWS Secrets Manager (Secrets Store CSI Driver) & AWS Certificate Manager** – TLS
  certificates, credentials, and license files mounted into pods.
- **AWS IAM Identity Center / Amazon Cognito & EKS Pod Identity** – OIDC for users and
  API callers; least-privilege pod access to AWS services.
- **Amazon CloudWatch (Container Insights)** – metrics, logs, and alarms.
- **AWS Backup** – backup of EKS cluster resources and EBS persistent volumes.
- **AWS KMS** – customer-managed keys for EBS, Kubernetes secrets, and backups.
- **Amazon GuardDuty (EKS Protection)** – runtime and audit-log threat detection.
- **AWS Direct Connect** – private connectivity from the customer network.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on AWS - EKS-v3.0.0.gif`
- `Adabas and Natural on AWS - EKS-v3.0.0.drawio` (editable source)

## Intended Use
Production, QA, and advanced development environments needing scalability,
zone-level resilience, API access to Natural business logic, and cloud-native
operations. For non-containerized deployments use the Basic or Resilient HA
blueprints.
