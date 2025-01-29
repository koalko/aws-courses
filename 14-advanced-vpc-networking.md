# Advanced VPC Networking

## VPC Flow Logs

Capture metadata (not contents). `VPC Flow Logs` monitors can be applied on three different levels:

- attached to a VPC (monitors all ENIs in that VPC)
- attached to a subnet (monitors all ENIs in that subnet)
- attached to a specific ENI(s)

Flow logs are **not realtime** (there is a delay). Log destinations: `S3` or `CloudWatch Logs`. You can also use `Athena` to query logs, stored in `S3`. Flow logs capture metadata from the capture point down: VPC -> Subnet(s) -> Interface(s). Flow logs can capture **accepted**, **rejected** or **all** metadata.

Flow log records have following fields:

- version
- account-id
- interface-id
- srcaddr
- dstaddr
- srcport
- dstport
- protocol (`1` for ICMP, `6` for TCP, `17` for UDP)
- packets
- bytes
- start
- end
- action
- log-status

The logs are not recorded for:

- `169.254.169.254` (metadata service)
- `169.254.169.123` (time server)
- DHCP
- Amazon DNS Server
- Amazon Windows License Server

## Egress-Only Internet Gateway

With IPv4 addresses are private or public. NAT allows private IPs to access public networks without allowing externally initiated connections. With IPv6 all IPs are public. Internet Gateway allows all IPv6 IPs **IN** and **OUT**.

Egress-Only is outbound-only for IPv6. Egress-Only Gateway is HA by default across all AZs in the region; scales as required. Default IPv6 route `::/0` added to route table with Egress-Only Internet Gateway as target.

## VPC Endpoints

### Gateway

Provide private access to `S3` and `DynamoDB`. Gateway endpoint is created per service / per region and associated with one or more subnets in a particular VPC. This causes a prefix list to be added to route table for these subnets (all traffic going to `S3`/`DynamoDB` redirected through Gateway endpoint as a target). Gateway endpoint is highly available across all AZs in a region by default. Endpoint policy is used to control what it can access (for example, only a subset of S3 buckets). Gateway endpoint is regional: can't access cross-region services. Gateway endpoint is only accessible from the associated VPC.

Using Gateway endpoints almost never require changes in the application.

Use cases:

- provide private VPC with access to `S3`/`DynamoDB`
  - Note: without Gateway endpoint private VPC can access `S3`/`DynamoDB` (as well as the rest of public internet) via NATGW
- private-only `S3` buckets

### Interface

Provide private access to AWS public services. Historically have been used to provide access to all services except of `S3` and `DynamoDB`, but `S3` is also supported now. Interface endpoints are added to a specific subnets (as `ENI`) and are not highly available. For `HA`: add one endpoint to one subnet per AZ used in VPC. Network access controlled via `Security Groups`. `Endpoint Policies` restrict what can be done (what can be accessed) with the endpoint. Only supports `TCP` protocol with `IPv4`. Uses `PrivateLink` (service which allow other services to be injected inside of VPC) under the hood.

Interface endpoint provides a new service endpoint DNS (for example: `vpce-123-xyz.sns.us-east-1.vpce.amazonaws.com`), which resolves to a private IP of an interface endpoint. There are several DNS names, which are given to each endpoint:

- regional DNS
- zonal DNS

Applications can optionally use these DNS names; or use PrivateDNS, which overrides the default DNS for services (using the private R53 hosted zone), which allow to access the service via original DNS.

Interface endpoint can be used, for example, to connect to a private EC2 instance (EC2 Instance Connect Endpoint).

## VPC Peering

Service, which allows to create a private and encypted link between **two** VPCs. Works same/cross-region and same/cross-account. Traffic transits over the AWS global network when using cross-region peering connections. Optionally public hostnames can resolve to private IPs. Same region security groups can reference peer security groups. VPC peering doesn't support transitive peering (A <-> B <-> C doesn't mean A <-> C automatically; you need a separate peer **per pair**).

When creating a VPC peering connection between two VPCs, a logical gateway object is created inside both of those VPCs. Routing configuration is needed to direct traffic flow for the remote CIDR at the peer gateway object. Security groups and NACLs can filter traffic.

VPC peering connections cannot be created where there is overlap in VPC CIDRs: ideally, never use overlapping address ranges in multiple VPCs.

When creating VPC peering connection through AWS console, it will be in the `Pending acceptance` state until the request will be manually accepted. After that, you need to:
- add a routes to route tables in **both** VPC with destination set to other VPC CIDR and target set to a proper peering connection
- add inbound rules to security groups, referencing other VPC security group or CIDR

## Quiz

- What is true of VPC Flow Logs (choose all that apply)
  - They capture packet metadata
  - They can be attached to a VPC
  - They can be attached to a subnet
  - They can be attached to an ENI
- To use S3 and DynamoDB in a private VPC which service is used
  - Gateway Endpoint
- To use SQS, SNS and Kinesis in a private VPC which service is used
  - Interface Endpoint
- Which service is used to provide outgoing only internet access to an IPv6 Instance
  - Egress-Only Internet Gateway
- To peer 4 VPC's how many peering connections are required
  - 6