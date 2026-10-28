# Adabas & Natural on AWS – Basic

## Overview
The Basic blueprint runs Adabas & Natural on Amazon EC2 instances. It is intended for
development, prototyping, and lift-and-shift workloads. Customers provision the AWS
resources shown in the diagrams and install the Adabas (data) and Natural
(application) components on those instances.

## What's New in v3.0.0
- **Natural OpenAPI Server** – exposes Natural subprograms as REST endpoints described
  by OpenAPI, so modern applications, portals, and integration platforms can call
  Natural business logic over HTTPS.
- **Natural Subprogram Server** – hosts and runs Natural subprograms on behalf of
  remote callers such as the Natural OpenAPI Server, separating API traffic from
  interactive Natural sessions.
- **Hub-and-spoke network** – a network hub VPC holds the shared ingress and inspection
  components (internal Application Load Balancer, AWS Network Firewall, VPC interface
  endpoints) and is connected to the workload VPC through AWS Transit Gateway. The
  regional services that serve the hub – Transit Gateway, AWS WAF, Route 53, Systems
  Manager Session Manager, and the optional Amazon API Gateway – are shown with it.

## Network Layout
- **Customer access:** all client traffic (HTTPS, and SSH / SSL/TLS for developers in
  the Development shape) arrives from the customer network over AWS Direct Connect or
  Site-to-Site VPN and the Transit Gateway. Nothing is exposed to the internet: the
  Application Load Balancer is internal and no Internet Gateway or public IPs are used.
- **Hub VPC:** internal Application Load Balancer with AWS WAF, AWS Network Firewall,
  and VPC interface endpoints shared through Route 53 private hosted zones.
- **Workload VPC:** private subnets only – one for the Development instance, or
  separate app and data subnets in the Distributed shape – each with a layer-scoped
  security group. An S3 gateway endpoint in the workload VPC carries backup traffic
  to Amazon S3 (gateway endpoints serve only their own VPC).

## Architecture Layers
- **API layer (Natural OpenAPI Server):** publishes selected Natural subprograms as
  OpenAPI-described REST services over **HTTPS**, secured with OIDC. Client traffic
  reaches it through the internal Application Load Balancer and AWS WAF in the hub VPC.
- **Application layer (Natural):** the Natural runtime and applications, plus the
  **Natural Subprogram Server** that runs the subprograms behind each API call.
  Reachable by end users on the Natural/application ports.
- **Data layer (Adabas):** the Adabas database and its files, held on dedicated
  Amazon EBS volumes.

The Basic blueprint supports three shapes:
- **Development – Single Instance:** all layers, including the Natural OpenAPI Server
  and the Natural Subprogram Server, co-located on one EC2 instance for simplicity.
- **Distributed:** the Adabas and Natural layers run on separate EC2 instances, each
  in its own private subnet, so each layer can be sized, secured, and maintained
  independently. The Natural OpenAPI Server and the Natural Subprogram Server run
  alongside Natural on the application instance.
- **Production – Single Instance:** a single hardened EC2 instance for small
  production footprints, with the API endpoint published only over HTTPS.

## Key New Products

### Natural OpenAPI Server
The Natural OpenAPI Server turns existing Natural subprograms into REST APIs without
changes to the Natural code.

- **API-first access:** each exposed subprogram becomes a REST endpoint with an
  OpenAPI specification that client teams can use to generate code and test.
- **No code changes:** existing business logic stays in Natural. The server maps the
  subprogram parameters to JSON request and response bodies.
- **Secure by default:** HTTPS for transport and OIDC for authentication, integrated
  with AWS IAM Identity Center, Amazon Cognito, or another OIDC provider.
- **Integration ready:** can sit behind Amazon API Gateway or another gateway for
  rate limiting, versioning, and a developer portal.

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
- **Amazon EC2** – compute hosting the API, application, and data layers.
- **Amazon EBS (gp3 / io2)** – persistent storage for Adabas data and Natural files;
  encrypted at rest with **AWS KMS** keys.
- **Hub & workload VPCs with AWS Transit Gateway** – private network boundary; the
  workload layers sit in private subnets of the workload VPC, the shared ingress and
  inspection components in the hub VPC.
- **Application Load Balancer (internal) + AWS WAF** – HTTPS entry point and web
  application firewall in front of the Natural OpenAPI Server.
- **AWS Network Firewall** – inspection of traffic between the customer network, the
  hub, and the workload VPC.
- **VPC endpoints** – interface endpoints in the hub VPC for AWS service APIs; an S3
  gateway endpoint in the workload VPC.
- **Security Groups** – layer-scoped firewalls. Allow HTTPS to the Natural OpenAPI
  Server only from the load balancer, and the Adabas ports only from the Natural
  layer. SSH and NDV (SSL/TLS) ports are opened only to developer ranges in the
  customer network, and only in the Development shape; administrators use Session
  Manager instead of SSH.
- **AWS Systems Manager Session Manager** – administrative access to the instances
  without bastion hosts, open SSH ports, or public IPs.
- **AWS IAM Identity Center / Amazon Cognito (OIDC)** – identity provider for API
  authentication; **AWS Directory Service** for LDAP/S where Natural Security uses it.
- **AWS Secrets Manager & AWS Certificate Manager** – TLS certificates and secrets
  used by the Natural OpenAPI Server.
- **IAM Roles (instance profiles)** – least-privilege access from instances to AWS
  services (e.g. Amazon S3 for backups, Secrets Manager for certificates).
- **AWS Direct Connect / Site-to-Site VPN** – private connectivity from the customer
  network into the Transit Gateway.
- **Amazon CloudWatch, AWS Backup, Amazon GuardDuty, VPC Flow Logs** – monitoring,
  backup, threat detection, and network logging.

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on AWS - Basic Development v3.0.0.gif`
- `Adabas and Natural on AWS - Basic Distributed v3.0.0.gif`

## Intended Use
Development, prototyping, API enablement pilots, and small production workloads. For
high availability and fault tolerance, use the Resilient HA blueprint; for
container-native scaling, use the Advanced blueprint.
