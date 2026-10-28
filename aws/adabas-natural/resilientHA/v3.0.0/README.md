# Adabas & Natural on AWS – Resilient HA

## Overview
The Resilient HA blueprint runs Adabas & Natural on AWS across three Availability
Zones for high availability and fault tolerance. It is intended for production
workloads. Customers provision the AWS resources shown in the diagram and install the
Adabas (data) and Natural (application) components on those EC2 instances.

## What's New in v3.0.0
- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI. In this blueprint it runs as multiple instances across Availability
  Zones behind an Application Load Balancer.
- **Natural Subprogram Server** – hosts and runs Natural subprograms on behalf of
  remote callers. It runs on its own instances in two zones, so API calls survive the
  loss of a single instance or zone.
- **Natural Availability Server** – multiple instances sharing session state through
  Amazon ElastiCache.
- **Separate endpoints per product** – the Application Load Balancer publishes one
  host name for the Natural OpenAPI Server and one for the Natural Availability
  Server, each with its own target group.
- **Hub-and-spoke network** – a network hub VPC holds the shared ingress and inspection
  components and is connected to the workload VPC through AWS Transit Gateway.

## Network Layout
- **Customer access:** client traffic arrives from the customer network over AWS
  Direct Connect or Site-to-Site VPN and the Transit Gateway. The Application Load
  Balancer is internal; no Internet Gateway or public IPs are used.
- **Hub VPC:** internal Application Load Balancer with AWS WAF, AWS Network Firewall,
  and VPC interface endpoints shared through Route 53 private hosted zones. Transit
  Gateway, AWS WAF, Route 53, Systems Manager Session Manager, and the optional Amazon
  API Gateway are regional services and are shown with the hub they serve.
- **Workload VPC:** three tiers of private subnets – web, app, and data – with one
  subnet per tier in each Availability Zone, each tier behind its own security group.
  An S3 gateway endpoint in the workload VPC carries backup traffic to Amazon S3.

## Architecture Layers
- **Ingress (hub VPC):** an internal Application Load Balancer with AWS WAF receives
  client HTTPS traffic and routes it by host name to two target groups:

  | Load balancer rule (example)     | Target group           | Targets                       |
  | -------------------------------- | ---------------------- | ----------------------------- |
  | `https://api.adanat.example.com` | `natural-openapi`      | Natural OpenAPI Server        |
  | `https://nas.adanat.example.com` | `natural-availability` | Natural Availability Server   |

  Each rule has its own TLS certificate (from AWS Certificate Manager), WAF rules,
  and health check, so API traffic and online user traffic can be secured, scaled,
  and monitored independently. The targets are registered by IP across the Transit
  Gateway. Replace the example host names with the customer's own DNS names;
  path-based rules on a single host name (for example `/api` and `/nas`) work the same
  way.
- **Web tier (private web subnets):** **Natural OpenAPI Server** and **Natural
  Availability Server** instances in all three zones. The Natural Availability Server
  instances keep session state in Amazon ElastiCache, so any instance can serve any
  user.
- **App tier (private app subnets):** an internal Network Load Balancer distributes
  calls from the web tier to:
  - **Natural Subprogram Server** instances (Natural Subprogram Server, Natural
    Security, Natural) in Zones A and B, serving the API calls.
  - **Natural Web/IO** instances (Natural Web/IO, Natural Security, Natural) in Zones
    A and B, serving the online applications.
  - an **Adabas Manager** instance in Zone C for database administration, reached by
    administrators through AWS Systems Manager Session Manager.
- **Data tier (private data subnets):** a 3-node **Adabas cluster** with one EC2
  instance per Availability Zone (primary in Zone A, secondaries in Zones B and C).
  Each instance runs Adabas with AEL and the Adabas REST Server on its own encrypted
  EBS volumes (gp3 / io2) for ASSO/DATA/WORK. The primary handles reads and writes
  over **ADATCP/S** and synchronizes to the secondaries, which serve reads.
- **Natural Batch:** runs on each Adabas instance. The instance in Zone A is the
  **primary** and runs batch jobs against the Adabas primary; the instances in Zones B
  and C are **failover** instances that take over if the primary or its zone is lost.

## Key New Products

### Natural OpenAPI Server
The Natural OpenAPI Server turns existing Natural subprograms into REST APIs without
changes to the Natural code.

- **API-first access:** each exposed subprogram becomes a REST endpoint with an
  OpenAPI specification that client teams can use to generate code and test.
- **Active-active instances:** each API call is handled independently, so every
  instance in the target group can serve traffic at the same time.
- **Zone resilience:** instances in three Availability Zones keep the API available
  if a zone is lost.
- **Secure by default:** HTTPS for transport and OIDC for authentication, integrated
  with AWS IAM Identity Center, Amazon Cognito, or another OIDC provider.
- **Integration ready:** can sit behind Amazon API Gateway for rate limiting,
  versioning, and a developer portal.

### Natural Subprogram Server
The Natural Subprogram Server is the runtime that runs Natural subprograms requested
by remote callers.

- **Redundant execution:** instances in two zones behind the internal Network Load
  Balancer, so the API layer always has a healthy target in another zone.
- **Isolation from online users:** API-driven calls run in their own server
  processes, separate from interactive Natural sessions.
- **Elastic capacity:** add instances or server processes to scale API throughput
  without changing the application.
- **Consistent security:** Natural Security rules apply to subprograms run through the
  server, just as they do for online users.

### Why It Matters
With the Natural OpenAPI Server and the Natural Subprogram Server deployed across
zones, Natural business logic can be offered as highly available APIs alongside the
existing online applications. The same resilient application and data layers serve
both channels.

## Cloud & Security Resources
- **Amazon EC2 (Multi-AZ)** – web, application, and data instances distributed across
  three Availability Zones.
- **Application Load Balancer (internal) + AWS WAF** – HTTPS entry point in the hub VPC; TLS
  termination and web application firewall in front of the Natural OpenAPI Server and
  Natural Availability Server instances.
- **Network Load Balancer (internal)** – distributes calls from the web tier to
  healthy Natural Subprogram Server and Natural Web/IO instances.
- **AWS Transit Gateway** – connects the hub VPC, the workload VPC, and the customer
  network (AWS Direct Connect / Site-to-Site VPN).
- **AWS Network Firewall** – inspection of traffic entering and leaving the workload
  VPC.
- **AWS Systems Manager Session Manager** – administrative access to Adabas Manager and
  the workload instances without bastion hosts or public IPs.
- **Amazon ElastiCache (Redis OSS / Valkey)** – shared session state for the Natural
  Availability Server instances.
- **Amazon API Gateway (optional)** – API gateway, throttling, and developer portal in
  front of the Natural OpenAPI Server.
- **Amazon EBS (gp3 / io2)** – encrypted persistent storage for each instance.
- **VPC & Subnets (Multi-AZ)** – private web, app, and data subnets in each zone.
- **Security Groups** – tier-scoped rules; HTTPS to the web tier only from the load
  balancer, Natural ports only from the web tier, Adabas ports only from the Natural
  instances.
- **Route 53 private hosted zones & VPC endpoints** – private name resolution;
  interface endpoints in the hub VPC for AWS service APIs, an S3 gateway endpoint in
  the workload VPC.
- **AWS IAM Identity Center / Amazon Cognito (OIDC)** – identity provider for API
  authentication.
- **AWS Secrets Manager & AWS Certificate Manager** – TLS certificates and secrets for
  the API layer.
- **IAM Roles (instance profiles)** – least-privilege access to AWS services (backups,
  monitoring, secrets).
- **AWS KMS** – customer-managed keys for EBS, S3, and secrets encryption.
- **AWS Backup, Amazon S3** – EBS snapshots and backup storage.
- **Amazon CloudWatch, VPC Flow Logs, Amazon GuardDuty** – metrics, logs, alarms,
  network logging, and threat detection.
- **Encryption** – encryption in transit and at rest across the API, application,
  and data layers.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on AWS - Resilient HA-v3.0.0.gif`
- `Adabas and Natural on AWS - Resilient HA-v3.0.0.drawio` (editable source)

## Intended Use
Production workloads requiring high availability, business continuity, disaster
recovery, and highly available API access to Natural business logic. For simpler
development setups use the Basic blueprint; for container-native scaling use the
Advanced blueprint.
