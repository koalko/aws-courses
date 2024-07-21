# Virtual Private Cloud (VPC) basics

## VPC Sizing and Structure

VPC Considerations
- VPC size
- any networks we can't use
- ranges other parties (VPC/cloud/on-premise/partner/vendor/etc.) use
- think about the future
- VPC structure: tiers and resiliency zones
- VPC minimum: /28 (16 IPs); maximum: /16 (65536 IPs)
- avoid common ranges
- reserve 2+ networks per region per account

### Analysis example

IP ranges to avoid
- 192.168.10.0/24
- 10.0.0.0/16
- 172.31.0.0/16
- 192.168.15.0/24
- 192.168.20.0/24
- 192.168.25.0/24
- 10.128.0.0/9 (10.128.0.0 - 10.255.255.255)

Let's assume we want to use 5 regions. Using 10.16 -> 10.127 total range. Start at 10.16 (US1), 10.32 (US2), 10.48 (US3), 10.64 (EU), 10.80 (Australia): each AWS account has 1/4th. `16` per VPC, 3 AZ (+1), 3 Tiers (+1): 16 subnets. `/16` split into 16 subnets = `/20` per subnet (4091 IPs).

## Custom VPCs

VPC is a regional service (all AZs in the region). Isolated network. Nothing goes in or out without explicit configuration. Allows flexible configuration (simple or multi-tier) and hybrid networking (other clouds and on-premises). There is a setting available upon creating the VPC to switch between default and dedicated tenancy. With dedicated tenancy any resource created inside VPC needs to be located on a dedicated hardware. VPC has 1 primary private IPv4 CIDR block with min `/28` (16 IPs) and max `/16` (65536 IPs). You can create optional secondary IPv4 blocks and optional single assigned IPv6 `/56` CIDR block. IPv6 addresses are all public, but you still need to explicitly allow connections.

VPC has DNS provided by Route53; DNS IP is `BaseIP + 2`. If `enableDnsHostnames` is enabled, instances have public DNS host names. If `enableDnsSupport` is enabled, DNS resolution in VPC works.

## VPC Subnets

VPC subnet is an AZ resilient sub-network of a VPC (within a particular AZ). Subnet can never be in multiple AZs. One subnet can be in one AZ; one AZ can have from 0 to ... subnets. Subnet's IPv4 CIDR is a subset of the VPC CIDR and cannot overlap with other subnets within the same VPC. Subnet can allocate an optional IPv6 CIDR (`/64` subset of `/56` VPC – space for 256).

Subnets can communicate with other subnets in the VPC.

### Subnet IP addressing

Reserved IP addresses (5 in total); for example, for `10.16.16.0/20` CIDR (`10.16.16.0` - `10.16.31.255` range):
- `10.16.16.0` - **network** address (starting address of the network)
- `10.16.16.1` - **network + 1** address (used by VPC router)
- `10.16.16.2` - **network + 2** address (reserved, used for DNS)
- `10.16.16.3` - **network + 3** address (reserved for future use)
- `10.16.31.255` - broadcast address (last IP in subnet)

VPC has a configuration object applied to it called DHCP (Dynamic Host Configuration Protocol) options set. DHCP allows devices to receive IP addresses automatically.

For each subnet two IP allocation options can be changed:
- auto assign public IPv4
- auto assign IPv6

### Example of multi-tier VPC subnets:

| NAME | CIDR | AZ | CustomIPv6Value |
|------|------|----|-----------------|
| sn-reserved-A | 10.16.0.0/20 | AZA | IPv6 00 |
| sn-db-A | 10.16.16.0/20 | AZA | IPv6 01 |
| sn-app-A | 10.16.32.0/20 | AZA | IPv6 02 |
| sn-web-A | 10.16.48.0/20 | AZA | IPv6 03 |
| | | | |
| sn-reserved-B | 10.16.64.0/20 | AZB | IPv6 04 |
| sn-db-B | 10.16.80.0/20 | AZB | IPv6 05 |
| sn-app-B | 10.16.96.0/20 | AZB | IPv6 06 |
| sn-web-B | 10.16.112.0/20 | AZB | IPv6 07 |
| | | | |
| sn-reserved-C | 10.16.128.0/20 | AZC | IPv6 08 |
| sn-db-C | 10.16.144.0/20 | AZC | IPv6 09 |
| sn-app-C | 10.16.160.0/20 | AZC | IPv6 0A |
| sn-web-C | 10.16.176.0/20 | AZC | IPv6 0B |

Remember to enable auto assign ipv6 on every subnet you create.

## VPC Routing, Internet Gateway & Bastion Hosts

Every VPC has a VPC router: a highly available device, which moves traffic. The router has a network interface in every subnet of a VPC under the `network + 1` address. By default routes traffic between subnets. Controlled by route tables (each subnet has one). A VPC has a main route table (used as a default subnet route table). A subnet can have only one route table associated with it, but a route table can be associated with any number of subnets.

In route tables, destination ("what traffic do I match") is usually a CIDR where higher prefix means higher route priority. Target means "where to send" and it can be either "IGW" or "local" (destination is in VPC).

IGW (Internet Gateway) is a region resilient gateway attached to a VPC. 1 VPC can have 0 or 1 IGWs attached; 1 IGW can be attached to 0 or 1 VPC. IGW runs from the AWS Public Zone and gateways the traffic between the VPC and the Internet or AWS Public Zone (S3, SQS, SNS, etc).

To use IGW (and make corresponding subnet **public**):
- create IGW
- attach IGW to VPC
- create custom RT (route table)
- associate RT
- add default routes (`0.0.0.0/0`, `::/0`) targeting IGW
- configure subnets to allocate IPv4 addresses

**Important note:** if you assign a public IP to EC2 instance, the association is maintained by the IGW; instance itself (OS) knows nothing about its public IP (it only knows a private IP). IGW replaces the private IPs with public IPs in the outgoing packets and public IPs with private IPs in the incoming packets. On the other hand, IPv6 addresses are all publicly routable, so IGW just forwards the IPv6 packets without any changes to source/destination.

Bastion Host ("Jumpbox") is an instance in a public subnet of a VPC. Processes incoming management connections to allow indirect access to internal VPC resources. Often the only way **in** to a VPC.

## Stateful vs Stateless Firewalls

Stateless firewall doesn't know anything about the state of a connection. For example, rules for a request part and a response part of the single connection needs to be configured separately for such firewall (2 rules: inbound and outbound). Considering the fact that clients usually pick ephemeral ports, corresponding rules tends to be wildcard-ish, which is not ideal.

Stateful firewall is able to determine that a request/response pair belong to the single connection. So, with stateful firewall you're only need to configure one (outbound or inbound) rule per connection; and the corresponding counterpart  will work automatically. This reduces admin overhead and chance of mistakes.

## Network Access Control Lists (NACLs)

NACL is basically a firewall available in AWS VPC. NACL is associated with a subnet. Every subnet inbound and outbound connection is affected by the NACL, but internal connections (between the instances within subnet) **are not**. Each NACL contains two sets of rules: inbound and outbound (referring to traffic direction). NACLs are stateless (so, each connection requires 1 outbound and 1 inbound rule). NACL rules match the IP range, port range and protocol. Each rule can state the explicit Allow or explicit Deny. Rules are processed in order (lowest number first). Once a match occurs, processing stops. `*` is an implicit Deny if nothing else matches.

A VPC is created with a default NACL: inbound and outbound rules have the implicit `Deny *`, and a higher-priority `Allow` all rule. So, **all traffic is allowed** by default.

Custom NACLs are created for a specific VPC and are initially associated with no subnets. Inbound and outbound rules have the implicit `Deny *`, so all traffic is disallowed by default.

NACLs only operates on IPs/CIDRs, ports and protocols; not on logical resources. NACLs can only be assigned to subnets. NACLs are often used with security groups to add explicit `Deny` (for example, for bot nets).

Each subnet have one NACL: default or custom. Each NACL can be associated with multiple subnets.

## Security Groups (SG)

SGs are stateful: they detect response traffic automatically, so no need to configure rules for the ephemeral ports.

SGs doesn't have an explicit `Deny`: only `Allow` or implicit `Deny`. So, SGs can't block specific bad actors.

SGs operates above NACLs, on the OSI 7 layer. SGs supports IPs/CIDRs/ports, and also supports logical resources – including other SGs and even itself.

SGs are attached to specific elastic network interfaces (ENIs).

SGs have inbound and outbound rules, just as NACLs.

## Network Address Translation (NAT) & NAT Gateway

NAT is a set of processes remapping source or destination IPs. IGW performs a type of NAT called static NAT (converting private IP to public IP and back).

IP masquerading means hiding CIDR blocks behind one IP, which gives private CIDR range **outgoing** internet access.

NAT Gateway resides in the public subnet; and the private subnet with instances, which requires access to the internet, routes all non-VPC traffic to the NAT Gateway through route table rules. And the NAT Gateway routes all the incoming traffic through the public subnet route table rules to the IGW.

NAT Gateway records all the important packets information (IP addresses, port numbers, etc.) into a translation table. After that NAT Gateway replaces the packet's private IP address with its own IP address.

If you need to give an instance its own public IPv4 address, then the only IGW is required. If you want to give private instances outgoing access to the internet and the AWS public zone services (for example, S3), then you need both the NAT Gateway and the IGW.

To reiterate, NAT Gateway:
- runs from a public subnet
- uses Elastic IP (static public IPv4)
- is an AZ resilient service (HA in that AZ), **not regionally resilient**
- for region resilience you need to:
  - deploy NAT GW in each AZ
  - have a route table for private subnets in each AZ with that NAT GW as a target
- is a managed service, scales to 45Gb/s, costs per duration and data volume

To use EC2 instance as a NAT GW, you need to disable `Source/Destination Checks` feature for that instance.

### Deciding between managed NAT GW and NAT Instance

If you value availability, bandwidth, low levels of maintenance and high performance, then you should choose NAT GW. NAT Instance can be cheaper and also can be used as a Bastion Host. Also, NAT Instances are just an EC2 instances, so you can filter traffic with NACLs and with SGs. You can **only use NACLs** with NAT GW.

NAT is not required for IPv6. The main focus of NAT is to allow private IPv4 addresses to be used to connect in an outgoing only way to the AWS public zone and public internet. Inside AWS all IPv6 addresses are publicly routable. IGW works with all IPv6 addresses directly. NAT GWs **don't work with IPv6**. `::/0` route pointed at the IGW as a target will give the instance bidirectional connectivity to the public internet and AWS public zone.

To make NAT GW work:
- create NAT GW in each AZ
- create route table in each AZ with the rule which routes all `0.0.0.0/0` traffic to a corresponding NAT GW
- associate route tables with private subnets in corresponding AZ

## Quiz

- What service does a VPC provide
  - Isolated Network
- What are true about default VPCs and custom VPCs (choose all that apply)
  - Regions can only have 1 default VPC and many custom VPCs
  - Custom VPCs allow flexible network configuration, the default VPC has a fixed scheme
  - Some services can behave oddly if the default VPC doesn't exist
  - Default VPCs can be recreated
- What are the valid sizes of a VPC
  - Max /16 & Min /28
- What is the Minimum and Maximum Size of a VPC Subnet
  - Min /28 & Max /16
- What is true for a VPC Subnet & AZ
  - An AZ can have many subnets, a subnet is in one AZ
- How many IP addresses are reserved in each VPC Subnet
  - 5
- How can an Internet Gateway (IGW) be configured to be highly available
  - It's HA by default - attached to a VPC
- What is true about SG's and NACLs (choose 2)
  - SGs can only ALLOW traffic
  - NACLs can ALLOW and DENY traffic
- What function does NAT serve
  - Allows IPv4 private instances outgoing access to the internet
- What is true of Route Tables and VPC Subnets (choose two)
  - A subnet can have one Route table attached
  - A route table can be associated with multiple subnets
