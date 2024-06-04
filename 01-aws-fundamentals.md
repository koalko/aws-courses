# AWS Fundamentals

## Public/private services

Three main zones (from the networking perspective):
- "public internet" zone (internet services)
- "AWS public" zone (public AWS services like IAM, S3, Route 53)
- "AWS private" zone (VPCs)
  - services from VPCs can be accessed from public zones via IGW

## Global infrastructure

AWS Regions: geographically/geopolitically separated full deployments of AWS resources. Example of a region name: `Asia Pacific (Sydney)` and a region code: `ap-southeast-2`.

AWS Availability Zones (AZs): each AWS region can have up to 6 of these. AZs are isolated from each other within one zone. AWS doesn't provide exact info on AZs locations. VPC, for example, could span across several AZs to provide resilience. Example of an AZ codes: `ap-southeast-2a`, `ap-southeast-2b`, `ap-southeast-2c`.

AWS Edge Locations: local distribution points, only contains CDN services and edge computing services.

See https://aws.amazon.com/about-aws/global-infrastructure/.

Service resilience types:
- Globally resilient (replicated across several AWS regions)
- Region resilient (replicated across AZs)
- AZ resilient (only run in a single AZ)

## VPC (Virtual Private Cloud)

A virtual network inside AWS.

A VPC is within 1 account & 1 region.

Private and isolated (unless you decide otherwise).

Two VPC types:
- default VPC
  - created by AWS
  - pre-configured
  - one per region
- custom VPCs
  - created and configured by user
  - private and isolated by default

VPC CIDR range controls the IP address range for the VPC. Example: `172.31.0.0/16`.

VPC can be divided in **subnets**. Each subnet resides in a specific AZ (set on creation and cannot be changed). Default VPC is always pre-configured to have one subnet in each AZ within the region. Each subnet uses part of VPC CIDR range. Example: `172.31.0.0/20`, `172.31.16.0/20`, `172.31.32.0/20`. These CIDR sub-ranges cannot overlap.

### Default VPC specifics

There can be one default VPC per region. It can be removed and recreated (for example, using `Actions` -> `Create default VPC` from `Your VPCs` page in AWS console). Default VPC CIDR range is always `172.31.0.0/16`. There is always a `/20` subnet in each AZ in the region. The following resources also created with the default VPC:
- Internet Gateway (IGW)
- Security Group (SG)
- Network Access Control List (NACL)
Subnets of a default VPC assign public IPv4 addresses.

## Elastic Compute Cloud (EC2) basics

IAAS (infrastructure as a service), provides virtual machines (instances).

Private service by-default (uses VPC).

EC2 is AZ resilient (instance fails if AZ fails).

Different instance sizes and capabilities.

On-demand billing (per second / per hour):
- for running the instance
- for the storage
- for any commercial software on the instance

Some options for the EC2 storage type:
- local on-host storage
- Elastic Block Store (EBS) - network storage service

Main instance resources:
- CPU
- memory
- storage (disk)
- networking

Main states of an instance lifecycle:
- running (charged for CPU, memory, storage, networking)
- stopped (charged only for storage)
- terminated (no charge)

Amazon Machine Image (AMI) can be used as a template to create an EC2 instance; AMI also can be created from an EC2 instance. AMI contains:
- attached permissions
  - public (everyone allowed)
  - owner (implicit allow)
  - explicit (specific AWS accounts allowed)
- root volume (to boot the OS)
- Block Device Mapping (to determine volumes for the OS; links volumes and device IDs)

Unrelated: 3389 is a default port for the Remote Desktop Protocol (RDP).

## S3 basics

S3 is a global storage platform. It is regional based / resilient. Public service, unlimited data, multi-user. Can host images, movies, audio, text, large data sets, ... S3 has objects (~ files) and buckets (containers for objects).

Object contains of:
- key (unique per bucket; similar to filename)
- value (data, file contents): from 0 bytes to 5TB
- version ID
- metadata
- access control
- subresources

S3 Buckets created in a particular region. Data never leaves region (unless transferred). Bucket identified by a **globally unique** name. Bucket can hold unlimied number of objects. Bucket has a flat structure (not a tree). But UI presents objects with `/` in names to look like they are organized in a tree. These "folders" often named "prefixes".

Bucket name restrictions:
- 3-63 characters, all lowercase, no underscores
- start with a lowercase letter or a number
- can't be formatted as an IP address
- bucket number limits per account:
  - soft: 100
  - hard: 1000

S3 is an object store; not file or block. You can't mount an S3 bucket as a drive. S3 is great for a large scale data storage, distribution or upload. S3 can be used as an input or output to many AWS products.

## CloudFormation basics

Infrastructure automation service.

Supports YAML & JSON templates.

Important template's sections:
- `Resources`: (the only mandatory part) list of resources to create/update/(remove)
- `Description`: free-format text description; needs to directly follow `AWSTemplateFormatVersion` if present
- `Metadata`: mainly controls the UI, but also responsible for some other things (?)
- `Parameters`: fields that prompts the user for more information
- `Mappings`: lookup tables
- `Conditions`: decision making; setting up the conditions to use those in `Resources`
- `Outputs`: the displayable results of applying the template (for example, an Instance ID for the created resource)

Resources inside the template are called **logical resources** (example: `Instance`). Logical resources have a `Type` (example: `AWS::EC2::Instance`). Logical resources also usually have `Properties` (example: `ImageId`).

Using the template, CloudFormation creates a **stack** (of logical resources). One template can create many stacks. Using the stack, CloudFormation creates (updates/removes) actual physical resources per stack's logical resources.

When the stack is deleted, CloudFormation removes all the corresponding physical resources.

When you upload the template via CloudFormation, it uploads it into the automatically created (`cf-templates-...` prefixed) S3 bucket.

## CloudWatch basics

CloudWatch main job is to collects and manage operational data (data, generated by environment).

CloudWatch parts:
- metrics
- logs
- events

CloudWatch allows to collect metrics, monitor metrics and execute actions based on metrics. Metrics are a data related to AWS products, applications, on-premise systems (examples: CPU utilization of an EC2 instance, disk space usage on an on-premise server, number of visitors per second on a website). CloudWatch is a public service. Some metrics are gathered natively (by default), usually these are related to AWS products. To gather metrics manually (for example, on an on-premise server), you need to install the CloudWatch agent.

CloudWatch allows to collect logs, monitor logs and execute actions based on logs. Examples of logs: Windows events logs, webserver logs, firewall logs, Linux server logs.

CloudWatch allows to generate events (and execute actions based on these events):
- in response to certain AWS Services state changes
- based on a time schedule

Namespace is a container for a monitoring data. All AWS data goes into the `AWS/<service>` namespace (for example, `AWS/EC2`). You can't use `AWS` namespace.

Metric is a collection of related data points in a time-ordered structure. Examples: CPU utilization, network in/out, disk utilization. Metric in itself is not identifying the source.

Datapoint – a single measurement of a specific metric: timestamp + value.

Dimensions separate datapoints for different things or perspectives within the same metric (for example, to differentiate the metrics for different instances within the `AWS/EC2` namespace). Think of extra parameters or tags.

Alarms are created for a specific metric. Alarm can execute a specific action when metric starts (or stops) satisfying the specified criteria. Possible alarm states:
- OK
- Alarm
- Insufficient data

## Shared Responsibility Model

General responsibility levels list:
- interface
- application
- data
- runtime
- container
- o/s
- hypervisor
- servers
- infrastructure
- facilities

Different approaches to hosting:
- on-premises (all responsibility levels are on customer)
- DC hosted (only **facilities** are covered by vendor)
- IaaS (levels up to **hypervisor** are covered by vendor)
- PaaS (levels up to **runtime** are covered by vendor)
- SaaS (all levels are covered by vendor)

Customer is responsible for security *in* the cloud:
- client-side data encryption, integrity & authentication; server-side encryption (file system and/or data); networking traffic protection (encryption, integrity, identity)
- operating system, network & firewall configuration
- platform, applications, identity & access management
- customer data

AWS is responsible for security *of* the cloud:
- regions, availability zones, edge locations
- hardware / AWS global infrastructure
- compute, storage, database, networking
- software

## High-Availability vs Fault-Tolerance vs Disaster Recovery

High-Availability (HA) aims to ensure an agreed level of operational performance, usually uptime, for a higher than normal period.

Uptime examples:
- 99.9% (three 9's) = 8.77 hours/year downtime
- 99.999% (five 9's) = 5.26 minutes/year downtime

Fault-Tolerance (FT) is the property that enables a system to continue operating properly in the event of the failure of some (one or more faults within) of its components.

Disaster Recovery is a set of policies, tools and procedures to enable the recovery or continuation of vital technology infrastructure and systems following a natural or human-induced disaster.

## Route53 (R53) Fundamentals

DNS-as-a-service. Global service, single database. Globally resilient. Two main sub-services:
- register domains
- host zones (managed nameservers)

To be able to register domains, R53 have connections with all the organizations managing TLDs (`.com`, `.io`, `.net`, `.org`). When registering the domain, R53 creates the zone file for the domain and allocates nameservers for this hosted zone (usually 4 servers for a zone file). Usually, TLD company uses NS records to refer to R53 nameservers for a particular zone.

Hosted zone can be public or private (linked to VPC). Hosted zone stores recordsets.

Record types:
- A: host to IPv4
- AAAA: host to IPv6
- CNAME: host to host
- MX: used to identify the host to use for sending emails (via SMTP); contains a priority and a host name
- TXT: arbitrary text; often used to prove domain ownership

DNS TTL (time to live): how long (in seconds) will DNS Resolver keep the DNS record in cache until invalidation. DNS Resolver can override TTL.
