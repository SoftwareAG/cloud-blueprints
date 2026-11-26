# Adabas & Natural on AWS – Advanced (EKS)

## Overview

The Advanced blueprint runs Adabas & Natural as containers on Amazon Elastic
Kubernetes Service (EKS) across three Availability Zones in one region, in a
hub-and-spoke network (hub VPC and workload VPC connected through AWS Transit
Gateway). It is intended for scalable, resilient production workloads on Kubernetes.
It is based on the v3.0.0 Resilient HA reference architecture, with the EC2 instances
replaced by Kubernetes workloads, the web and app tiers running in all three zones
instead of two, and Natural Batch running as a Deployment that Kubernetes reschedules. Customers provision the EKS cluster and supporting
resources shown in the diagram and deploy the official Software AG Adabas and Natural
container images.

## What's New in v3.0.0

- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI, deployed as replicas in all three zones.
- **Natural Subprogram Server** – runs Natural subprograms on behalf of the Natural
  OpenAPI Server, deployed as its own Deployment behind its own Service.

## Reference Architecture

### Ingress (hub VPC)

An internet-facing Application Load Balancer with AWS WAF receives public HTTPS
traffic from browser, mobile, and API clients; internal and admin traffic comes in
privately over Direct Connect / Site-to-Site VPN. Through the AWS Load Balancer
Controller the ALB publishes two endpoints, each routed to its own Kubernetes
Service:

| ALB endpoint (example)     | Kubernetes Service     | Backend pods                |
| -------------------------- | ---------------------- | --------------------------- |
| `https://api.example.com`  | `natural-openapi`      | Natural OpenAPI Server      |
| `https://nas.example.com`  | `natural-availability` | Natural Availability Server |

Each endpoint has its own listener rule, TLS certificate (from ACM), WAF rules, and
health check, so API traffic and online user traffic can be secured, scaled, and
monitored independently. Replace the example host names with the customer's own DNS
names; path-based routing on a single host name (for example `/api` and `/nas`) works
the same way.

The ALB uses IP targets: with the Amazon VPC CNI, pods have VPC IP addresses, so the
ALB in the hub VPC reaches them directly across the Transit Gateway.

### Web tier (`adanat-web` namespace, zones A–C)

- **Natural OpenAPI Server** and **Natural Availability Server (NAS)** Deployments,
  one replica per zone, each behind its own Service.
- NAS keeps Natural session affinity in Amazon ElastiCache (Valkey / Redis OSS,
  Multi-AZ), so a user's session survives the loss of a NAS pod or zone.

### App tier (`adanat` namespace, zones A–C)

Two internal Services (ClusterIP) connect the web tier to the app tier:

- **`natural-subprogram`** → **Natural Subprogram Server** pods (Natural Subprogram
  Server, Natural Security, Natural), serving calls from the Natural OpenAPI Server.
- **`natural-webio`** → **Natural Web I/O Server** pods (Natural Web I/O Server,
  Natural Security, Natural), serving the online applications behind NAS.

Both run one replica per zone, so every tier keeps a healthy target if a zone is lost.

### Data tier (`adanat` namespace, zones A–C)

- **Adabas Cluster** – a 3-replica **StatefulSet** (`adabas-0`, `adabas-1`,
  `adabas-2`), one pod per zone. Each pod runs Adabas with AEL and the Adabas Rest
  Server, and has its own zonal EBS persistent volume (gp3 / io2, shared-nothing).
- **Primary and secondaries:** one pod is the primary and takes all read/write (R/W)
  calls over **ADATCP/S**; the other two are secondaries that also serve reads.
  **SYNC replication** runs between all three pods. The diagram shows `adabas-0` as
  the current primary, but the role moves on failover; Natural's Adabas client
  locates the primary, so Natural reconnects without configuration changes.
- **Natural Batch** – a single-replica Deployment (or a CronJob for scheduled jobs)
  that connects to the primary through the Adabas client. It is not pinned to a zone;
  if its pod, node, or zone fails, Kubernetes reschedules it elsewhere.
- **Adabas Manager** – a pod that monitors and administers all three Adabas pods. It
  is not published through the ALB.

### Administration

- **Private EKS API endpoint** – the cluster API is reachable only from the private
  network. Administrators connect through **AWS Systems Manager Session Manager** (for
  example to a small admin instance with `kubectl`) or over Direct Connect / VPN; the
  CI/CD pipeline uses a runner inside the workload VPC.
- **Adabas Manager access** – via `kubectl port-forward`, or through an internal load
  balancer reachable only over Direct Connect / VPN.

### Namespaces and placement

- **`adanat-web`** – the web-facing tier; the only namespace exposed through the ALB.
- **`adanat`** – the Natural and Adabas core: Natural Subprogram Server, Natural Web
  I/O Server, Natural Batch, Adabas Manager, and the Adabas cluster StatefulSet. It
  is reachable only from `adanat-web` and from administrators.
- Pods are spread across zones with topology spread constraints and anti-affinity, so
  the loss of a zone leaves the API, online, and database services running.
- Network policies allow only `adanat-web` pods to reach the Natural Services in
  `adanat`, and only Natural pods and Adabas Manager to reach the Adabas pods.

## Key Components on EKS

### Natural OpenAPI Server

- Turns existing Natural subprograms into OpenAPI-described REST endpoints without
  code changes.
- Exposed through its own Service (`natural-openapi`) and ALB endpoint.
- Runs active-active: each API call is handled independently, so every replica can
  serve traffic. Scale with replicas or a Horizontal Pod Autoscaler.
- HTTPS at the ALB. The server does not authenticate callers itself: it passes the
  caller's OIDC bearer token (from IAM Identity Center or another OIDC provider) with
  each call to the Natural Subprogram Server.

### Natural Subprogram Server

- Runs the Natural subprograms requested through the Natural OpenAPI Server, in its
  own Deployment, separate from interactive Natural workloads.
- One replica per zone behind the `natural-subprogram` Service.
- Authenticates each API call: validates the caller's token passed on by the Natural
  OpenAPI Server and runs the subprogram as the impersonated Natural user, with
  Natural Security rules applied.

### Natural Availability Server

- Web front end for Natural online applications, running as replicas behind its own
  Service (`natural-availability`) and ALB endpoint.
- Natural session affinity kept in Amazon ElastiCache, so a lost pod or zone does not
  interrupt users.
- Pods can be drained and updated one by one for maintenance without an outage.

### Adabas Cluster (StatefulSet)

- Three pods, one per zone, each with its own persistent volume and independent copy
  of the data – no shared disk.
- SYNC replication between all three pods; automatic failover to a secondary if the
  primary pod, node, or zone is lost.
- Rolling maintenance one pod at a time, and read scale-out on the secondaries.
- Data encrypted at rest by Adabas Encryption for Linux (AEL) and KMS-encrypted EBS.

## Cloud & Security Resources

### Network hub – VPC & regional services

- **Application Load Balancer + AWS WAF (internet-facing, AWS Load Balancer
  Controller)** – ingress, TLS termination, and web application firewall, with
  separate rules for the Natural OpenAPI Server and NAS endpoints.
- **AWS Network Firewall** – EKS outbound traffic is routed through it with route
  tables; also controls traffic between the hub, the workload VPC, and the internet.
- **AWS Transit Gateway + Site-to-Site VPN** – connects the customer network (with
  Direct Connect) and the hub and workload VPCs.
- **Amazon Route 53** – public hosted zone for the ALB host names; private hosted
  zones for internal names and VPC endpoints.
- **Amazon API Gateway (optional)** – throttling, versioning, and usage plans in front
  of the Natural OpenAPI Server.
- **VPC endpoints** – interface endpoints for ECR, Secrets Manager, KMS, SSM, STS, and
  CloudWatch, and a gateway endpoint for S3.
- **AWS Systems Manager Session Manager** – admin access without bastion hosts or
  public IPs.

### Workload VPC – EKS

- **Amazon EKS** – private API endpoint, Amazon VPC CNI, managed node groups across
  three Availability Zones.
- **Kubernetes Services & network policies** – ClusterIP Services between tiers;
  policies as described above.
- **Security groups** – on the node groups and the ALB (security groups for pods
  where needed).

### Shared services

- **AWS Secrets Manager (CSI Secrets Store)** – TLS material, credentials, and licence
  files mounted into pods.
- **Amazon S3** – Adabas-level backups and file exchange.
- **Amazon ElastiCache (Valkey / Redis OSS, Multi-AZ)** – Natural session affinity for
  NAS.
- **Amazon ECR** – Software AG container images, pulled through a VPC endpoint.
- **CloudWatch Container Insights & Amazon CloudWatch** – metrics, logs, and alarms.
- **AWS Backup (EKS + EBS)** – backup of cluster resources and persistent volumes.
  Volume snapshots of a running database are only crash-consistent, so also take
  Adabas-level backups to S3.
- **AWS KMS** – customer-managed keys for EBS, S3, and secrets encryption.
- **Amazon EBS (gp3 / io2, zonal)** – persistent volumes for the Adabas pods.
- **Amazon GuardDuty (EKS Protection)** – runtime and audit-log threat detection.

### Tools & identity

- **IaC (Terraform / CloudFormation)** – provisions the AWS resources.
- **AWS CodePipeline (GitOps / Helm)** – deploys the Kubernetes workloads; use a
  runner inside the workload VPC, as the EKS API is private.
- **IAM Identity Center (OIDC)** – identity provider for users and API callers.
- **EKS Pod Identity** – least-privilege pod access to AWS services.

## Regional Redundancy (optional)

The diagram covers one region: it protects against the loss of a pod, a node, or an
Availability Zone, not a whole region. Customers who need regional-level redundancy
can deploy a second EKS cluster in another region from the same IaC and GitOps
definitions, and restore it from AWS Backup copies and Adabas backups replicated to
that region (S3 cross-region replication). Agree recovery time and recovery point
objectives with the business.

## Architecture Diagrams

See the `docs` folder:

- `Adabas and Natural on AWS - Advanced EKS v3.0.0.gif` – the reference architecture.

Abbreviations: AEL – Adabas Encryption for Linux; NAS – Natural Availability Server.

## Intended Use

Production, QA, and advanced development environments needing scalability,
zone-level resilience, API access to Natural business logic, and Kubernetes-based
operations. For non-containerized deployments use the Basic or Resilient HA
blueprints.
