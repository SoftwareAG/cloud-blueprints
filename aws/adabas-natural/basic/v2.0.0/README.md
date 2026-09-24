# Adabas & Natural on AWS – Basic

## Overview
The Basic blueprint runs Adabas & Natural on AWS EC2 instances. It is intended for
development, prototyping, and lift-and-shift workloads. Customers provision the AWS
resources shown in the diagrams and install the Adabas (data) and Natural
(application) components on those instances.

## Architecture Layers
- **Application layer (Natural):** the Natural runtime and applications. Reachable
  by end users on the Natural/application ports.
- **Data layer (Adabas):** the Adabas database and its files, held on dedicated
  Amazon EBS volumes.

The Basic blueprint supports three shapes:
- **Development – Single Instance:** both layers co-located on one EC2 instance for
  simplicity.
- **Distributed:** the Adabas and Natural layers run on separate EC2 instances so
  each layer can be sized, secured, and maintained independently.
- **Production – Single Instance:** a single hardened instance for small production
  footprints.

## Cloud & Security Resources
- **Amazon EC2** – compute hosting the application and data layers.
- **Amazon EBS** – persistent block storage for Adabas data and Natural files;
  enable encryption at rest.
- **VPC & Subnets** – private network boundary; place each layer in its own subnet
  where the layers are separated.
- **Security Groups** – layer-scoped firewalls. Restrict SSH to trusted ranges and
  allow the Adabas ports only from the Natural layer.
- **IAM Roles** – least-privilege access from instances to AWS services (e.g. S3
  for backups).

## Architecture Diagrams
See the `docs` folder:
- `Adabas and Natural on AWS Basic Setup Development Single Instance_v2.0.0.gif`
- `Adabas and Natural on AWS Basic Setup Distributed_v2.0.0.gif`
- `Adabas and Natural on AWS Basic Setup PROD Single Instance _v2.0.0.gif`

## Intended Use
Development, prototyping, and small production workloads. For high availability and
fault tolerance, use the Resilient HA blueprint; for container-native scaling, use
the Advanced blueprint.
