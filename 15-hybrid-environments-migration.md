# Hybrid environments and migration

## Border Gateway Protocol (BGP) 101

Routing protocol. BGP as a system is made up of lots of self-managing networks known as Autonomous System (`AS`). BGP views AS as a blackbox and only concerns itself with network routing in and out of AS. Each AS has a unique number (`ASN`, `0`-`65535`), which is allocated by IANA. The range `64512`-`65534` is private. There are also 32-bit ASNs. BGP operates over `tcp/179`, it's reliable (error correction / flow control), but not automatic (peering is manually configured).

BGP is a `path-vector` protocol: it exchanges the best path to a destination between peers. The path is called the `ASPATH`. So, it is a BGP responsibility to build up the network topology map and allow the exchange between different AS's.

BGP sub-types:

- iBGP: internal BGP; routing within an AS
- eBGP: external BGP; routing between AS's

### Example

- Brisbane
  - ASN: 200
  - CIDR: 10.16.0.1/16
  - Connected to:
    - Adelaide: 1 Gbps Fibre
    - Alice Springs: 5 Mbps Satellite
- Adelaide
  - ASN: 201
  - CIDR: 10.17.0.1/16
  - Connected to:
    - Brisbane: 1 Gbps Fibre
    - Alice Springs: 1 Gbps Fibre
- Alice Springs
  - ASN: 202
  - CIDR: 10.18.0.1/16
  - Connected to:
    - Brisbane: 5 Mbps Satellite
    - Adelaide: 1 Gbps Fibre

Route table for Brisbane:

| Destination    | Next Hop    | ASPATH          |
| -------------- | ----------- | --------------- |
| `10.16.0.0/16` | `0.0.0.0`   | `i`             |
| `10.17.0.0/16` | `10.17.0.1` | `201`,`i`       |
| `10.18.0.0/16` | `10.18.0.1` | `202`,`i`       |
| `10.18.0.0/16` | `10.17.0.1` | `201`,`202`,`i` |

Route table for Adelaide:

| Destination    | Next Hop    | ASPATH          |
| -------------- | ----------- | --------------- |
| `10.17.0.0/16` | `0.0.0.0`   | `i`             |
| `10.16.0.0/16` | `10.16.0.1` | `200`,`i`       |
| `10.18.0.0/16` | `10.18.0.1` | `202`,`i`       |
| `10.18.0.0/16` | `10.16.0.1` | `200`,`202`,`i` |

Route table for Alice Springs:

| Destination    | Next Hop    | ASPATH          |
| -------------- | ----------- | --------------- |
| `10.18.0.0/16` | `0.0.0.0`   | `i`             |
| `10.16.0.0/16` | `10.16.0.1` | `200`,`i`       |
| `10.17.0.0/16` | `10.17.0.1` | `201`,`i`       |
| `10.16.0.0/16` | `10.17.0.1` | `201`,`200`,`i` |

Note: `i` is the origin.

BGP exchanges the shortest ASPATH between peers. Brisbane -> Alice Springs would default to the satellite link, even though the longer fibre would provide better performance. AS Path Prepending can be used to artificially make the sattelite path look longer making the fibre path preferred. Example for Brisbane -> Alice Springs:

| Destination    | Next Hop    | ASPATH                |
| -------------- | ----------- | --------------------- |
| `10.16.0.0/16` | `0.0.0.0`   | `i`                   |
| `10.17.0.0/16` | `10.17.0.1` | `201`,`i`             |
| `10.18.0.0/16` | `10.17.0.1` | `201`,`202`,`i`       |
| `10.18.0.0/16` | `10.18.0.1` | `202`,`202`,`202`,`i` |

In summary, an `AS` will advertise all the shortest paths it knows to all its peers; the `AS` prepends its own `ASN` onto the path: this creates a source to destination path, which BGP routers can learn and propagate.

## IPSec VPN Fundamentals

IPSec is a group of protocols which work together. Their aim is to set up secure networking tunnels across insecure networks: for example, connecting two secure networks (or more specifically their routers, called peers) across the public internet. IPSec provides authentication. Traffic is encrypted.

**Interesting traffic** is traffic which matches certain rules; if data matches these rules, the VPN tunnel is created to carry traffic through to its destination. If there is no interesting traffic, tunnels are eventually torn down.

IPSec has two main phases:

- `IKE (Internet Key Exchange) Phase 1` (slow and heavy)
  - Two versions: 1 (older) and 2 (newer, extra features)
  - Authenticate with pre-shared key (password) / certificate
  - Using asymmetric encryption to agree on and create a shared symmetric key
  - End result: IKE SA created (phase 1 tunnel)
- `IKE Phase 2` (fast and agile)
  - Uses the keys agreed on in phase 1
  - Agree encryption method and keys used for bulk data transfer
  - End result: IPSec SA created (phase 2 tunnel, which architecturally runs over phase 1)

Phase 2 tunnel can be torn down and re-created while Phase 1 tunnel is still in place.

Example of IPSec flow:

- Phase 1
  - Two peers authenticate with pre-shared key or certificate (confirming identity)
  - Diffie–Hellman (DH) key exchange:
    - Each side creates a DH private key, which is used to decrypt the data and sign things
    - Each side uses private key to derive the corresponding public key
    - Public keys are exchanged over the public internet
    - Each side takes their private key and the remote peers public key; and intependently generates the same shared DH key
  - Exchange key material and agreements using DH key
  - Each side uses DH key + the exchanged key material to generate a final phase 1 symmetrical key
  - This key is used to encrypt everything passing the Phase 1 tunnel (`IKE Security Association`)
- Phase 2
  - Symmetrical key is used to encrypt and decrypt agreements and pass more key material between the peers
  - Best shared encryption and integrity methods communicated and agreed
  - DH key and exchanged key material are used to create symmetrical IPSec key (designed for large data transfer)
  - IPSec key is used for bulk encryption and decryption of interesting traffic
  - Across the Phase 1 tunnel you have the pair of security associations (in both directions)

Two different types of IPSec VPN:

- `Policy-based VPNs`
  - rule sets match traffic
  - different rules / security settings for different types of traffic
  - multiple SA pairs: one phase 1 tunnel, running multiple phase 2 tunnels, based on policies
  - more complicated to set up
- `Route-based VPNs`
  - target matching based on prefix
  - single SA pair: one phase 1 tunnel, running two phase 2 tunnel, based on routes
  - less functionality, simplier to set up

## AWS Site-to-Site VPN

A logical connection between a VPC and on-premises network encrypted using IPSec, running over the public internet. Full HA – if you design and implement it correctly. Quick to provision (less than an hour).

Components:

- `VPC`
- Virtual Private Gateway (`VGW`)
  - lives in AWS public zone
  - has public endpoints in different AZs (HA)
- Customer Gateway (`CGW`)
  - lives in a customer network
  - can be HA: several separate `CGW`s connects to different `VGW` endpoints
- VPN connection between `VGW` and `CGW`

### Static and dynamic VPN

Static VPN:

- Routes for remote site added to route tables as **static routes**.
- Networks for remote side **statically** configured on the **VPN connection**.
- Simple.
- No load balancing and multi-connection failover.

Dynamic VPN:

- Uses `BGP` (Borderline Gateway Protocol), which allow routers to exchange network information.
- `BGP` is configured on both the customer and and AWS side using `ASN`. Networks are exchanged via `BGP`.
- Multiple VPN connections provide HA and traffic distribution.
- Routes for remote side can still be added to route tables as static routes.
- `Route propagation` can be enabled for VPC route tables, which allow routes to be added to route tables automatically.

VPN considerations:

- VPN speed cap is **~1.25 Gbps**.
- Latency might be an issue (public internet is inconsistent).
- Cost: AWS hourly cost, GB out cost, data cap (on premises).
- Speed of setup: hours.
- Can be used as a backup for Direct Connect (`DX`).
- Can be used **with** Direct Connect (`DX`).

## Direct Connect (DX)

### Concepts

A physical connection (1, 10 or 100 Gbps) into an AWS region:

```
Business Premises -> DX Location -> AWS Region
```

You actually order **port allocation** at a DX Location.

You pay for:

- port hourly cost
- outbound data transfer

Provisioning time can be months: you need to lay a physical cable from your location to DX Location. Also, no resilience (cable cutting shuts down the connection).

Provides low and consistent latency + high speeds. Can be used to access AWS private services (VPCs) and AWS public services (not public internet).

DX Location usually is not owned by AWS; often it is a large regional data center, within which AWS rents space. Data center incorporates AWS Direct Connect Cage, which contains one or more AWS DX Routers (DX Endpoints). You can also rent space and have a Customer (or Comms Partner) Cage, which might contain Customer (or Partner) DX Router. Direct connection between AWS Port and Customer (Partner) Port called `Cross Connect`.

### Resilience

You can assume that the connection between DX Location(s) and AWS Region is always HA (redundant high speed connections). Within the DX Location, AWS DX Router is connected to a Customer (Provider) DX Router by a single Cross-Connect connection by default. A DX is extended from the DX Location to a Customer Premises. There might be a lot of single points of failure:

- DX Location
- DX Router
- Cross Connect
- Customer DX Router
- Extension
- Customer Premises
- Customer Router

This can be improved by multiplying each component mentioned above (including Customer Premises and DX Locations).

### Public VIF (virtual interface) + VPN (encryption)

Provides an encrypted and authenticated tunnel with low and consistent latency. Uses a public VIF and VGW/TWG public endpoints. Transit agnostic (DX / public internet). End-to-end encrypted (CGW <-> TGW/VGW). Has wider vendor support than MACsec, but also has more cryptographic overhead (limits speeds). Can be used while DX is being provisioned and/or as a DX backup.

VPNs traditionally operate via the public internet into the external side of the AWS Public Zone. **VPN+DX** accesses the same public VPN endpoints using a **Public VIF**. The VPN tunnel creates a private (VPC) connection between the customer gateway and an AWS VGW or TGW. It uses a public VIF, but creates a private connection into a VPC.

## Transit Gateway (TGW)

Network Transit Hub to connect VPCs to on-premises networks. Significantly reduces network complexity. Single network object: HA and scalable. **Attachments** (`VPC`, `Site-to-Site VPN`, `Direct Connect Gateway`) to other network types.

To route traffic between multiple VPCs and a single CGW without the TGW, VPCs needs to be interconnected (6 VPC peering connections) and also each VPC needs to be connected to CGW (8 VPN tunnels, 2 per VPC). This architecture will be HA on AWS side, but CGW will be a single point of failure. To make the architecture fully HA, it will require another CGW on a customer side, doubling the required number of VPN tunnels. Such full mesh network is complex, requires a lot of admin overhead and scales poorly.

With the same setup (4 VPCs, 2 CGWs), adding a TGW with Site-to-Site VPN attachment will require only 4 VPN tunnels (2 per CGW): TGW becomes the AWS side termination endpoint. With TGW you can also create VPC attachments (configured with a subnet in each AZ where service is required), so TGW acts like a highly available inter-VPC router. This architecture can also easily be expanded by adding a peering attachment (Cross-Region, Same/Cross-Account).

Supports transitive routing (single TGW with multiple attachments instead of manually setting up mesh network). Can be used to create global networks. Can be shared between AWS accounts using AWS RAM. Peer with different regions (same- or cross-accounts). In general, using TGW often reduces network' complexity (sometimes significantly).

## Storage Gateway

Storage Gateway normally runs as a virtual machine (or a hardware appliance) on-premises. Acts as a bridge between the storage and AWS. Presents storage using `iSCSI`, `NFS` or `SMB`. Integrates with `EBS`, `S3` and `Glacier` within AWS. Used for migrations (from on-premises into AWS), extensions (of a data center into AWS), storage tiering, DR, replacement of backup systems.

It is common to have Network Attached Storage (on-premises), which is being used by a bunch of servers (on-premises) with own local disks, via the iSCSI protocol (presents raw block storage over the network as block devices to servers).

### Volume

Volume Gateway works in two modes: cached and stored.

#### Stored mode

The virtual appliance presents volumes over iSCSI to servers running on-premises (similar to how NAS hardware does it). These volumes consume capacity on-premises, so the Storage Gateway has `Local Storage`: primary storage location for all of the volumes that storage gateway presents out over iSCSI. So, in stored mode everything is stored locally. There is also a separate area called Upload Buffer. All data written to Local Storage, also written to this area temporarily, and then its copied asynchronously to AWS via the Storage Gateway Endpoint (public endpoint), and then its copied to S3 in the form of EBS snapshots.

This architecture:

- great for full disk backups of servers
- can assist with disaster recovery (you can quickly create EBS volumes from S3 EBS snapshots)
- doesn't improve datacenter capacity (main copy of data is stored on the gateway)

Limitations:

- 32 volumes per gateway
- 16 TB per volume
- 512 TB per gateway

#### Cached mode

Same basic architecture as in Stored mode, but the primary data location is in an **AWS-managed area of S3** (only visible from the storage gateway console). There is a `Cache Storage` instead of `Local Storage` on-premises. Only frequently accessed data stored locally in this architecture. Storage appears on-premises, but is actually in AWS. This mode provides capacity extension. To summarize: data stored in AWS and cached on-premises.

Limitations:

- 32 volumes per gateway
- 32 TB per volume
- 1 PB per gateway

### Tape (VTL)

Large backups still often done on tape. Example: LTO-9 Media can hold 24 TB of raw data (up to 60 TB compressed). 1 tape drive can use 1 tape at a time. Tape loaders (aka tape robots or media changers) can swap tapes. A tape library is 1+ `drive`, 1+ `loader` and `slots`. `Shelf` usually means somewhere not in the library. Managing tape backups internally on an enterprise scale can be very costly. Storage Gateway in Tape mode is cheaper and much easier to setup.

Backup server interacts with on-premises Storage Gateway (Tape) through iSCSI (like a normal tape loader). On-premises Storage Gateway (Tape) implements upload buffer and local cache to store frequently used data. Storage Gateway AWS Endpoint presents two main capabilities:

- Virtual Tape Library (VTL), backed by S3
- Virtual Tape Shelf (VTS), backed by Glacier

Virtual tape can take from 100 GiB to 5 TiB (max size of S3 object). Total virtual library capacity: 1 PB across 1500 virtual tapes. Virtual shelf storage capacity is unlimited. Process of moving data from VTL to VTS called "Archive"; from VTS to VTL – "Retrieve".

### File

Bridges on-premises file storage and S3 by creating mount points (shares), available via NFS or SMB. Map directly onto an S3 bucket: files, stored into a mount point, are visible as objects in an S3 bucket. Read and write caching ensure LAN-like performance.

File Gateway usually runs as a virtual appliance on-premises and has a local storage, which is used for read/write caching. Each file share, created within File Gateway, is linked with an S3 bucket (this link called **bucket share**). S3 objects are visible as on-premises files.

You can have up to 10 bucket shares per File Gateway. Primary data is held in S3 (so you can use other AWS services to integrate with file processing; or, for example, use S3 storage classes/lifecycle functionality).

It is important to use `NotifyWhenUploaded` API to notify other gateways when objects are changed (in case if different gateways use the same bucket).

File Gateway doesn't support any form of locking (one update can override another). Either use read-only mode on other shares, or tightly control file access.

You can setup cross-region replication (CRR) of objects between a shared S3 bucket and a backup one (multi-regional DR).

## Snowball / Snowball Edge / Snowmobile

Designed to move large amounts of data in and out of AWS. Devices are physical storage units (sized from suitcase to truck). Can be ordered from AWS empty, load up, return; or ordered from AWS with data, empty, return.

### Snowball

Ordered from AWS, log a job, device delivered. Data is encrypted at rest using KMS. Supports 50 TB or 80 TB capacity and 1 Gbps or 10 Gbps network. 10 TB to 10 PB economical range (multiple devices). Storage-only device.

### Snowball Edge

Like Snowball, but provides both storage and compute. Supports larger capacity. Supports faster network: 10 Gbps (RJ45), 10/25 (SFP), 45/50/100 Gbps (QSFP+).

Three options:

- storage optimized: 80 TB, 24 vCPU, 32 GiB RAM, 1 TB SSD (with EC2)
- compute optimized: 100 TB + 7.68 TB NVME, 52 vCPU, 208 GiB RAM
- compute with GPU: as compute optimized + GPU

Ideal for remote sites or where data processing on ingestion is needed.

### Snowmobile

Portable data center within a shipping container on a truck. Special order. Ideal for single location when 10+ PB is required. Supports up to 100 PB per snowmobile. Not economical for multi-site (unless huge) or sub 10 PB.

## Directory Service

Provides a managed directory: a store of users, objects and other configuration. Stores objects (Users, Groups, Computers, Servers, File Shares) with a structure (domain/tree). Multiple trees can be grouped into a forest. Commonly used in Windows environments. Sign-in to multiple devices with the same username/password provides centralized management for assets.

Types:

- Microsoft Active Directory Domain Services (AD DS)
- SAMBA (opensource implementation of AD)

Directory Service is an AWS managed implementation of AD, which runs within a VPC. Can be HA when deployed into multiple AZs. Some AWS services (for example, `Amazon Workspaces`) **need** a directory.

Directory can be:

- isolated
- integrated with existing on-premises system
- act as a proxy back to on-premises (connector mode)

### Simple AD mode

Standalone directory which uses SAMBA 4 (open source). Can operate in two different sizes:

- small: up to 500 users
- large: up to 5000 users

Integrates with AWS services:

- `EC2` instances can join Simple AD
- `Workspaces` can use Simple AD for logins and management

Not designed to integrate with any existing on-premises directory system such as Microsoft AD.

### AWS Managed Microsoft AD

AWS Managed Microsoft AD can have a secure connection (VPN/DX) to on-premises directory. Primary location is in AWS. Trust relationships can be created between AWS and on-premises directory systems. Resilient if the VPN fails; services in AWS will still be able to access the local directory running in Directory Service.

Full Microsoft AD DS Running in 2012 R2 Mode. Supports Microsoft AD aware applications running in AWS.

### AD Connector

Applicable when you want to use some AWS service (for example, Workspaces), which requires AD; you already have on-premises directory, and you don't want to create an AWS managed AD. AD Connector allow AWS services to interact with on-premises directory through private connection (VPN/DX).

In other words: primary directory is located on-premises; requests from AWS are proxied back to the existing directory. If private connectivity fails, the AD proxy won't function, interrupting service on AWS side.

### Picking between Directory Service modes

- `Simple AD`
  - default
  - simple requirements
  - directory in AWS
- `Microsoft AD`
  - applications in AWS which needs MS AD DS
  - or you need to trust AD DS
- `AD Connector`
  - use AWS Services which need a directory without storing any directory info in the cloud
  - proxy to your on-premises directory

## DataSync

Data Transfer service **to** and **from** AWS. Used for migrations, data processing transfers, archival / cost effective storage, DR/BC. Designed to work at huge scale: each agent can handle 10 Gbps data transfer. Each job can handle 50 million files. Keeps metadata (permissions/timestamps). Includes built-in data validation.

Supports:

- scaling: 10 Gbps per agent (~100 TB per day)
- bandwidth limiters (to avoid link saturation)
- incremental and scheduled transfer options
- compression and encryption
- automatic recovery from transit errors
- AWS service integration: S3, EFS, FSx
- pay as you use (per GB cost for data moved)

The DataSync agent runs on a virtualization platform such as VMWare and communicates with the AWS DataSync Endpoint (using encryption in-transit via TLS). For example, agent can be running on-premises, interacting with NAS storage via NFS or SMB.

Schedules can be set to ensure the transfer of data occurs during or avoiding specific time periods. Customer impact can be minimized by setting a bandwidth limit (in MiB/s).

DataSync components:

- `task`: a job within DataSync; defines what is being synced, how quickly, from where, and to where
- `agent`: software used to read or write to on-premises data stores using NFS or SMB
- `location`: every task has two locations (**from** and **to**); examples: Network File System (NFS), Server Message Block (SMB), EFS, FSx, S3

## FSx for Windows File Server

Provides fully managed native Windows file servers (represented as file shares). Designed for integration with Windows environments. Integrates with Directory Service or Self-Managed AD (on-premises). Can be deployed into Single or Multi-AZ within a VPC. Can perform on-demand and scheduled backups. Accessible using VPC, Peering, VPN, Direct Connect.

Native Windows file system; supports de-duplication (sub file), Distributed File System (DFS), KMS at-rest encryption and enforced encryption in-transit. File shares accessed via SMB. Supports volume shadow copies (file level versioning).

Performance:

- 8 MB/s - 2 GB/s
- 100k's IOPS
- less than 1ms latency

Key features and benefits:

- VSS: user-driven restores
- native file system accessible over `SMB`
- Windows permissions model
- supports DFS (Distributed File System): scale-out file share structure
- managed: no file server admin
- integrates with AWS DS or your own directory

## FSx For Lustre

Lustre file system was designed for High Performance Computing (HPC). FSx for lustre is a managed Lustre file system, which supports Linux clients with POSIX-style permissions. Designed for use cases like machine learning, big data, financial modelling. Can scale to hundreds of GB/s throughput and offers sub-millisecond latency.

Deployment types:

- `Scratch`
  - optimized for high-end performance
  - short-term storage
  - no replication or HA
- `Peristent`
  - longer term
  - replication (in **one AZ**)
  - self-healing when hardware failure occurs

You can backup (manual or automatic) to S3 for both deployment types.

Accessible over `VPN` or `DX`.

When creating, you can associate file system with repository (S3 bucket). Data will be lazily loaded from S3 into the file system as it's needed. There is no automated backwards sync, so S3 bucket only used as a foundation. But you can export data back to S3 at any point using `hsm_archive`.

Data elements:

- metadata stored on Metadata Targets (MST)
- objects are stored on object storage targets (OSTs) (1.17 TiB each)

Baseline performance is based on size. Minimum size is 1.2 TiB; then increments of 2.4 TiB used. For `Scratch` baseline performance is 200 MB/s per TiB of storage. `Persistent` offers 50 MB/s, 100 MB/s and 200 MB/s per TiB of storage. Burst up to 1,300 MB/s per TiB (credit system).

## AWS Transfer Family

Managed file transfer service. Supports transferring to or from `S3` and `EFS`.

Provides managed "servers" which support various protocols:

- File Transfer Protocol (FTP): unencrypted file transfer
- File Transfer Protocol Secure (FTPS): file transfer with TLS encryption
- Secure Shell (SSH) File Transfer Protocol (SFTP): file transfer over SSH
- Applicability Statement 2 (AS2): structured B2B data

Transfer Family supports a wide variety of identity providers:

- Service Managed
- Directory Service
- Custom (Lambda/APIGW)

Supports Managed File Transfer Workflows (`MFTW`): serverless file workflow engine.

Transfer Family Endpoint types:

- Public
  - SFTP only
  - dynamic IP, managed by AWS (use DNS)
  - can't control access
- VPC - Internet
  - SFTP / FTPS
- VPC - Internal
  - FTP / SFTP / FTPS
  - static private IP + Elastic IP (static public IP)
  - SG and NACL can be used to control access

AWS Transfer Family is multi-AZ. Cost is based on provisioned server per hour + data transferred. With FTP and FTPS only Directory Service or Custom IDP are supported. FTP can be used in VPC only (cannot be public). AS2 protocol supported in VPC internet/internal.

AWS Transfer Family can be used if you need access to S3/EFS, but with existing protocols (potentially integrating with existing workflows, or using `MFTW` to create new ones).

## Quiz

- Which storage gateway mode can replace a tape drive with S3 storage
  - VTL
- Which storage gateway mode can be used to present storage over SMB to clients
  - File
- Which storage gateway mode is good for data centre extension into AWS
  - Volume - gateway cached
- What storage product in AWS can be used for windows environments for shared storage
  - FSx
- Is Direct Connect Encrypted
  - No
- Can a private encrypted connection be created using Direct Connect? (if so how)
  - Yes by using a Site-to-Site VPN over a public VIF
- What protocol does Site-to-Site VPN use
  - IPSec
- When should a direct connect be used
  - When low latency is important
  - When high throughput is important
- What modes does the AWS Directory Service Support
  - Simple Active Directory
  - Active Directory Connector
  - AWS Managed Microsoft AD
- Which mode should directory service be run in when you have an existing active directory on-premises and want a minimal AWS footprint to run isolated services which need a directory.
  - Active Directory Connector
