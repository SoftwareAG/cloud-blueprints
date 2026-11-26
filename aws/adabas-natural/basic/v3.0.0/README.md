# Adabas & Natural on AWS – Basic

## Overview

The Basic blueprint runs Adabas & Natural on Amazon EC2 in a hub-and-spoke network:
a hub VPC for shared connectivity and security services, connected through AWS
Transit Gateway to a workload VPC. It covers development, prototyping, test stages
(INT, UAT, QA), and lift-and-shift workloads that do not need high availability.
Customers provision the AWS resources shown in the diagrams and install the Adabas
(data) and Natural (application) components on those instances.

## What's New in v3.0.0

- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI, so modern applications, portals, and integration platforms can call
  Natural business logic over HTTPS.
- **Natural Subprogram Server** – hosts and runs Natural subprograms on behalf of
  remote callers such as the Natural OpenAPI Server, separating API traffic from
  interactive Natural sessions.

## Reference Architectures

The Basic blueprint has two variants. Both use the same hub-and-spoke network and the
same shared services; they differ in how many instances run the workload and how
users reach it.

### Development – Single EC2 Instance

A workspace for developers and internal teams. Every layer runs on one EC2 instance.

- **Development EC2 instance:** Natural OpenAPI Server, Natural Subprogram Server,
  Natural, and the Natural Development Server (NDV) alongside Adabas (with AEL), the
  Adabas Rest Server, and Adabas Manager. Adabas data sits on encrypted EBS gp3
  volumes, which are enough for development I/O at lower cost.
- **No public ingress:** there is no load balancer. Internal users and testers call
  the Natural OpenAPI Server over HTTPS, developers use terminal emulation over SSH
  and NaturalONE (NJX, VS Code) over SSL/TLS to NDV – all privately through AWS Direct
  Connect / Site-to-Site VPN.
- **No OIDC:** API access is limited by the private network and security groups.
- **Admin access:** AWS Systems Manager Session Manager – no bastion host, open SSH
  port from the internet, or public IP.
- Not for production.

### Distributed – App EC2 Instance and Data EC2 Instance

The Adabas and Natural layers run on separate instances, so each can be sized,
secured, and maintained on its own. Suited to INT, UAT, and QA, or to production where
HA is not required.

- **App EC2 instance:** Natural OpenAPI Server with the Natural Subprogram Server
  behind it; Natural Availability Server (NAS) with the Natural Web I/O Server behind
  it; Natural Security and the Natural runtime underneath both.
- **Data EC2 instance:** Adabas (with AEL), Adabas Rest Server, Adabas Manager, and
  Natural Batch. Adabas data sits on encrypted EBS volumes (gp3, or io2 for heavy
  I/O). Natural Batch needs its own Natural installation and licence on this instance.
- **Public ingress:** an internet-facing Application Load Balancer with AWS WAF in the
  hub VPC is the public HTTPS entry point for browser, mobile, and API clients, in
  front of the Natural OpenAPI Server and NAS (IP targets across the Transit Gateway).
  API callers present OIDC bearer tokens from IAM Identity Center or another OIDC
  provider, which the Natural Subprogram Server validates.
- **Private traffic:** internal and admin traffic comes in through Direct Connect /
  Site-to-Site VPN.
- **Inside the App EC2 instance:** the Natural OpenAPI Server calls the Natural
  Subprogram Server, and NAS calls the Natural Web I/O Server (Natural internal calls).
- **App to data:** the App EC2 instance reads and writes Adabas over **ADATCP/S**.
- **Admin access:** SSM Session Manager to both instances – the App instance for
  patching, Natural deployment, and NAS / Web I/O configuration; the Data instance for
  Adabas Manager.
- **Single instance:** every layer runs once. NAS keeps session state in memory, so no
  external session store is needed. Each instance sits in its own private subnet with
  a layer-scoped security group.

## Key New Products

### Natural OpenAPI Server

The Natural OpenAPI Server turns existing Natural subprograms into REST APIs without
changes to the Natural code.

- **API-first access:** each exposed subprogram becomes a REST endpoint with an
  OpenAPI specification that client teams can use to generate code and test.
- **No code changes:** existing business logic stays in Natural. The server maps the
  subprogram parameters to JSON request and response bodies.
- **Secure transport and identity:** HTTPS for transport. The server does not
  authenticate callers itself: it passes the caller's OIDC bearer token (from IAM
  Identity Center or another OIDC provider) with each call to the Natural Subprogram
  Server (used in the Distributed variant).
- **Integration ready:** can sit behind Amazon API Gateway for throttling, versioning,
  and usage plans.

### Natural Subprogram Server

The Natural Subprogram Server is the runtime that runs Natural subprograms requested
by remote callers.

- **Dedicated execution:** runs API-driven subprogram calls in their own server
  processes, separate from interactive Natural sessions.
- **Direct data access:** runs on the Natural application layer and reaches Adabas
  over **ADATCP/S**, like any other Natural workload.
- **Independent sizing:** the number of server processes can be tuned to API demand
  without affecting online users.
- **Authentication and impersonation:** validates the caller's token passed on by the
  Natural OpenAPI Server and runs each subprogram as the impersonated Natural user,
  with Natural Security applied.

## Cloud & Security Resources

### Network hub – VPC & regional services

- **Application Load Balancer + AWS WAF (internet-facing)** – Distributed only; the
  public HTTPS entry point in front of the Natural OpenAPI Server and NAS.
- **AWS Network Firewall** – inspects and controls traffic between the hub, the
  workload VPC, and the internet.
- **AWS Transit Gateway + Site-to-Site VPN** – connects the customer network (with
  Direct Connect) and the hub and workload VPCs.
- **AWS Systems Manager Session Manager** – administrative access to the instances
  without bastion hosts or public IPs.
- **Amazon Route 53** – private hosted zones for internal names and VPC endpoints; the
  Distributed variant also needs a public hosted zone for the ALB hostname.
- **VPC endpoints** – interface endpoints for Secrets Manager, KMS, SSM, and CloudWatch,
  and a gateway endpoint for S3, so traffic to AWS services stays private.

### Workload VPC

- **Amazon EC2** – one instance (Development) or an App and a Data instance
  (Distributed), in one Availability Zone, in private subnets.
- **Security groups** – layer-scoped firewalls. Allow HTTPS to the Natural OpenAPI
  Server and NAS only from the ALB or approved clients, the Adabas ports only from the
  Natural layer, and no inbound SSH from the internet (SSM is used instead).

### Shared services

- **AWS Secrets Manager + ACM** – TLS certificates and secrets for the Natural OpenAPI
  Server and NAS.
- **Amazon S3** – backups and file exchange, reached through the S3 gateway endpoint.
- **Amazon CloudWatch & CloudWatch Logs** – metrics, logs, and alarms.
- **AWS Backup** – instance and EBS snapshot backup and restore.
- **Amazon EBS** – persistent storage for Adabas data and Natural files: gp3 in the
  Development variant, gp3 or io2 in the Distributed variant.
- **AWS KMS** – customer-managed keys for EBS, S3, and Secrets Manager encryption.
- **VPC Flow Logs** – Distributed only; network traffic logs.
- **Amazon GuardDuty** – threat detection.

### Tools & identity

- **IaC (Terraform / CloudFormation)** – provisions the resources above.
- **AWS CodePipeline** – Distributed only; build and release pipelines, deploying to
  the instances through SSM.
- **IAM Identity Center (OIDC)** – Distributed only; identity provider for API
  authentication.
- **AWS Directory Service (LDAP/S)** – directory for OS and Natural Security user
  authentication.
- **IAM roles** – least-privilege access from the instances to AWS services (e.g. S3
  for backups, Secrets Manager for certificates).

## Architecture Diagrams

See the `docs` folder:

- `Adabas and Natural on AWS - Basic Development v3.0.0.gif` – single EC2 instance,
  private access only.
- `Adabas and Natural on AWS - Basic Distributed v3.0.0.gif` – App and Data EC2
  instances, public HTTPS through an internet-facing ALB + AWS WAF.

Abbreviations: AEL – Adabas Encryption for Linux; NDV – Natural Development Server;
NAS – Natural Availability Server.

## Intended Use

| Variant      | Typical use                                        | Public ingress   | HA |
|--------------|----------------------------------------------------|------------------|----|
| Development  | Developer workspaces, prototyping, API pilots      | No               | No |
| Distributed  | INT, UAT, QA; production where HA is not required  | Yes (ALB + WAF)  | No |

For high availability and fault tolerance, use the Resilient HA blueprint; for
scaling on Kubernetes, use the Advanced blueprint.
