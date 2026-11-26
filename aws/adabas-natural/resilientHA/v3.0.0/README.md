# Adabas & Natural on AWS – Resilient HA

## Overview

The Resilient HA blueprint runs Adabas & Natural on Amazon EC2 spread across three
Availability Zones in one region, in a hub-and-spoke network (hub VPC and workload VPC
connected through AWS Transit Gateway). Every tier survives the loss of an instance or
a whole Availability Zone. It is intended for production workloads. Customers
provision the AWS resources shown in the diagram and install the Adabas (data) and
Natural (application) components on those instances.

## What's New in v3.0.0

- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI. In this blueprint it runs active-active in two Availability Zones behind
  the Application Load Balancer.
- **Natural Subprogram Server** – hosts and runs Natural subprograms on behalf of
  remote callers. It runs on an App instance in each of two zones behind an internal
  Network Load Balancer, so API calls survive the loss of an instance or a zone.

## Reference Architecture

### Web tier (web subnets, zones A and B)

- **Natural OpenAPI Server** – one Web EC2 instance per zone, active-active.
- **Natural Availability Server (NAS)** – one Web EC2 instance per zone. NAS keeps
  Natural session affinity in **Amazon ElastiCache** (Valkey / Redis OSS, Multi-AZ),
  so a session keeps working when it lands on the other NAS instance.
- **Ingress:** an internet-facing Application Load Balancer with AWS WAF (Multi-AZ) in
  the hub VPC is the public HTTPS entry point for browser, mobile, and API clients, in
  front of both services. API callers present OIDC bearer tokens from IAM Identity
  Center or another OIDC provider, which the Natural Subprogram Server validates. Internal and admin traffic from the customer
  network comes in privately over Direct Connect / Site-to-Site VPN.

### App tier (app subnets, zones A and B)

- **Internal Network Load Balancer** (cross-zone) – distributes calls from the web
  tier to healthy App instances in either zone.
- **App EC2 instances** – each runs Natural, Natural Security, and either the
  **Natural Subprogram Server** (behind the Natural OpenAPI Server) or the **Natural
  Web I/O Server** (behind NAS).

### Data tier (data subnets, zones A, B and C)

- **Adabas Cluster** – three Data EC2 instances, one per zone, each running Adabas
  (with AEL) and the Adabas Rest Server: one **primary** and two **secondaries**. The
  cluster uses a shared-nothing architecture: every node has its own encrypted EBS
  volumes (gp3 / io2), and **SYNC replication** runs between all three nodes, so every
  node holds the current data at all times.
- **Access:** the App instances send all read/write (R/W) calls to the primary over
  **ADATCP/S**; the two secondaries also serve read transactions.
- **Failover:** if the primary fails, a secondary takes over. Natural's Adabas client
  locates the new primary, so Natural reconnects without configuration changes.
- **Natural Batch** – installed on every Data instance but active on one node at a
  time (the primary's instance in the diagram); another node takes over on failover.
  Each Data instance needs a Natural installation and licence for this.

### Management (mgmt subnet, zone C)

- **Adabas Manager** – a dedicated EC2 instance that monitors and administers all three
  Adabas nodes. It has no public ingress and is not published through the ALB.
- **Admin access:** AWS Systems Manager Session Manager to every EC2 instance – no
  bastion hosts, jump hosts, open SSH ports, or public IPs.
- Adabas Manager is a single instance; it is not on the request path, so a zone C
  outage does not stop the application.

## Key New Products

### Natural OpenAPI Server

The Natural OpenAPI Server turns existing Natural subprograms into REST APIs without
changes to the Natural code.

- **API-first access:** each exposed subprogram becomes a REST endpoint with an
  OpenAPI specification that client teams can use to generate code and test.
- **Active-active instances:** each API call is handled independently, so every
  instance behind the ALB can serve traffic at the same time.
- **Zone resilience:** instances in two Availability Zones keep the API available if a
  zone is lost.
- **Secure by default:** HTTPS for transport. The server does not authenticate
  callers itself: it passes the caller's OIDC bearer token with each call to the
  Natural Subprogram Server.
- **Integration ready:** Amazon API Gateway can optionally be added in front for
  throttling, versioning, and usage plans (not shown in the diagram).

### Natural Subprogram Server

The Natural Subprogram Server is the runtime that runs Natural subprograms requested
by remote callers.

- **Redundant execution:** an instance runs on an App instance in each zone, so the
  API layer always has a healthy target.
- **Isolation from online users:** API-driven calls run in their own server
  processes, separate from interactive Natural sessions.
- **Elastic capacity:** add App instances or server processes to scale API throughput
  without changing the application.
- **Authentication and impersonation:** validates the caller's token passed on by the
  Natural OpenAPI Server and runs each subprogram as the impersonated Natural user.
- **Consistent security:** Natural Security rules apply to subprograms run through the
  server, just as they do for online users.

### Why It Matters

With the Natural OpenAPI Server and the Natural Subprogram Server deployed across
zones, Natural business logic can be offered as highly available APIs alongside the
existing online applications. The same resilient application and data tiers serve
both channels.

## Cloud & Security Resources

### Network hub – VPC & regional services

- **Application Load Balancer + AWS WAF (internet-facing, Multi-AZ)** – TLS termination
  and the public HTTPS entry point for the Natural OpenAPI Server and NAS (IP targets
  in the workload VPC across the Transit Gateway).
- **AWS Network Firewall** – inspects and controls traffic between the hub, the
  workload VPC, and the internet.
- **AWS Transit Gateway + Site-to-Site VPN** – connects the customer network (with
  Direct Connect) and the hub and workload VPCs.
- **Amazon Route 53** – public hosted zone for the ALB hostname; private hosted zones
  for internal names such as Adabas Manager and the VPC endpoints.
- **AWS Systems Manager Session Manager** – admin access to every instance.
- **VPC endpoints** – interface endpoints for Secrets Manager, KMS, SSM, and
  CloudWatch, and a gateway endpoint for S3. ElastiCache runs inside the workload VPC,
  so it needs no endpoint.

### Workload VPC

- **Amazon EC2 (multi-AZ)** – Web, App, and Data instances across zones, plus the
  Adabas Manager instance in zone C.
- **Subnets** – private web, app, data, and mgmt subnets in each zone.
- **Internal Network Load Balancer (cross-zone)** – web tier to app tier.
- **Security groups** – one per tier. HTTPS to the web tier only from the ALB, app tier
  only from the web tier, Adabas ports only from the app tier and Adabas Manager,
  Adabas cluster replication ports only between the three Data instances, and no
  inbound SSH (SSM is used instead).

### Shared services

- **AWS Secrets Manager + ACM** – TLS certificates and secrets for the web tier.
- **Amazon S3** – backups and file exchange.
- **Amazon ElastiCache (Valkey / Redis OSS, Multi-AZ)** – Natural session affinity for
  NAS.
- **Amazon CloudWatch & CloudWatch Logs** – metrics, logs, and alarms.
- **AWS Backup** – instance and EBS snapshot backup and restore.
- **Amazon EBS (gp3 / io2)** – per-node storage for Adabas data and Natural files.
- **AWS KMS** – customer-managed keys for EBS, S3, and Secrets Manager encryption.
- **VPC Flow Logs** – network traffic logs.
- **Amazon GuardDuty** – threat detection.

### Tools & identity

- **IaC (Terraform / CloudFormation)** – provisions the resources above.
- **AWS CodePipeline** – build and release pipelines, deploying to the instances
  through SSM.
- **IAM Identity Center (OIDC)** – identity provider for API authentication.
- **AWS Directory Service (LDAP/S)** – directory for OS and Natural Security user
  authentication.
- **IAM roles** – least-privilege access from the instances to AWS services.

## Regional Redundancy (optional)

The diagram covers one region: it protects against the loss of an instance or an
Availability Zone, not a whole region. Customers who need regional-level redundancy
can add **AWS Elastic Disaster Recovery** to replicate the instances to a second
region and fail over the whole stack there, together with cross-region AWS Backup
copies. Plan the recovery region's network, Route 53 records, and load balancer
alongside it, and agree recovery time and recovery point objectives with the business.

## Architecture Diagrams

See the `docs` folder:

- `Adabas and Natural on AWS - Resilient HA v3.0.0.gif` – the reference architecture.

Abbreviations: AEL – Adabas Encryption for Linux; NAS – Natural Availability Server.

## Intended Use

Production workloads requiring high availability and highly available API access to
Natural business logic, with optional regional redundancy through AWS Elastic Disaster
Recovery. For development and test stages, or production without HA, use the Basic
blueprint; for scaling on Kubernetes, use the Advanced blueprint.
