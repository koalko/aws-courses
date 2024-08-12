# Elastic Compute Cloud (EC2) basics

## Virtualization 101

EC2 provides virtualization as a service (IaaS product). Virtualization is a process of running more than one OS on a piece of physical hardware.

Virtualization types:
- Software Virtualization. Host OS is running on a hardware and contains a hypervisor.
  - Emulated Virtualization (Software Virtualization). Hypervisor performs binary translation, capturing all guest OSes system calls. Slow.
  - Para-Virtualization. Guest OSes are modified so system calls are replaced with user calls to the hypervisor (hyper calls).
- Hardware (Assisted) Virtualization. CPU is aware of virtualization. Hypervisor runs directly on hardware.
  - SR-IOV (single route IO virtualization). Different hardware pieces (for example, network card) become virtualization-aware. In `EC2` this is called `Enhanced Networking`.

## EC2 Architecture and Resilience

EC2 instances are virtual machines (OS + resources). EC2 instances run on EC2 hosts (hardware). Hosts can be shared (default, hardware is shared between customers, but instances are still isolated) or dedicated (you're paying for the entire host). EC2 is an AZ resilient service (AZ fails => host fails => instances fail). EC2 host is running within a single AZ. EC2 hosts have:
- CPU
- Memory
- Instance Store (local storage): temporary
- Networking
  - Storage (Elastic Block Store maps here)
  - Data (Elastic Network Interface maps here)

Instances stay on the host until one of two things happen:
- host fails (or taken down for maintenance)
- instance is stopped and then started (not just restarted)

EC2 and EBS are both AZ services. You cannot cross AZs with instances or with EBS volumes.

EC2 is good for:
- traditional OS + Application compute
- long-running compute
- server-style applications
- burst or steady-state load
- monolithic application stack
- migrated application workloads or disaster recovery

## EC2 Instance Types

EC2 instance type influences:
- raw CPU, memory, local storage capacity and type
- resource ratios
- storage and data network bandwidth
- system architecture / vendor
- additional features and capabilities

### EC2 categories

- `General Purpose`: (default) diverse workloads, equal resource ratio
- `Compute Optimized`: media processing, HPC, scientific modelling, gaming, machine learning
- `Memory Optimized`: processing large in-memory datasets, some database workloads
- `Accelerated Computing`: hardware GPU, field programmable gate arrays (FPGAs)
- `Storage Optimized`: sequential and random IO, scale-out transactional databases, data warehousing, Elasticsearch, analytics workload

### Decoding EC2 types

Example: `R5dn.8xlarge`.

- `R5dn.8xlarge` – instance type
  - `R` – instance family
  - `5` – instance generation
  - `dn` – additional capabilities
  - `8xlarge` – instance size

### Instance categories

- `General Purpose`
  - `A1`, `M6g`: Graviton (A1), Graviton 2 (M6g) ARM based processors. Efficient.
  - `T3`, `T3a`: Burst Pool. Cheaper assuming nominal low levels of usage, with occasional peaks.
  - `M5`, `M5a`, `M5n`: Steady state workload alternative to T3/T3a; Intel/AMD Architecture.
- `Compute Optimized`
  - `C5`, `C5n`: Media encoding, scientific modelling, gaming servers, general machine learning.
- `Memory Optimized`
  - `R5`, `R5a`: Realtime analytics, in-memory caches, certain DB applications (in-memory operations).
  - `X1`, `X1e`: Large scale in-memory applications; lowest $ per GiB memory in AWS.
  - `High Memory` (`u-Xtb1`): Highest memory of all AWS instances.
  - `z1d`: Large memory and CPU, with directly attached NVMe.
- `Accelerated Computing`
  - `P3`: GPU instances (Tesla v100 GPUs) – parallel processing and machine learning.
  - `G4`: GPU instances (NVIDIA T4 Tensor) – machine learning inference and graphics intensive.
  - `F1`: FPGAs – genomics, financial analysis, big data.
  - `Inf1`: Machine learning – recommendation, forecasting, analysis, voice, conversation.
- `Storage Optimized`
  - `I3`/`I3en`: Local high performance SSD (NVMe) – NoSQL databases, warehousing, analytics.
  - `D2`: Dense Storage (HDD) – data warehousing, HADOOP, Distributed File Systems, data processing – lowest price disk throughput.
  - `H1`: High Throughput, balance CPU/Memory – HDFS, MAPR-FS, file systems, Apache Kafka, big data.

### External information

[Official AWS documentation](https://aws.amazon.com/ec2/instance-types/) on EC2 instance types.

[EC2 instances list](https://instances.vantage.sh/).

## Storage

**Direct** (local) attached storage – storage on the EC2 host.

**Network** attached storage – volumes delivered over the network (EBS).

**Ephemeral** storage – temporary storage (example: Instance Store).

**Persistent** storage – permanent storage – lives on past the lifetime of the instance (example: Network Attached Store).

Storage types:
- **Block** storage: volume presented to the OS as a collection of blocks; no structure provided. Mountable. Bootable.
- **File** storage: presented as a file share; has structure. Mountable. Not bootable.
- **Object** storage: collection of objects, flat. Not mountable. Not bootable.

Storage performance:

`IO (block) Size` * `IOPS` = `Throughput`

## Elastic Block Store (EBS) Service Architecture

Block storage – raw disk allocations (volume) – can be encrypted using KMS. Instances see block device and create file system on this device (ext3/4, xfs). Storage is provisioned in one AZ (replicated and resilient in that AZ). Attached to one EC2 instance (or other service) over a storage network – **only in the same AZ**. Can be detached and reattached; not lifecycle linked to one instance; persistent. Volume can be snap-shotted into S3. Subsequently, volume can be created from an S3 snapshot. Supports different physical storage types, different sizes and different performance profiles. Billed based on GB-month (and in some cases performance).

## EBS Volume Types - General Purpose

### GP2

Volume size: 1GB-16TB.

Volume created with an IO credit. An IO credit is 16KB. IOPS assume 16KB. 1 IOPS is 1 IO in 1 second. IO "Credit" bucket capacity: 5.4M IO Credits. Fills at rate of Baseline Performance. Bucket fills with minimum 100 IO Credits per second regardless of volume size. Beyond the 100 minimum the bucket fills with 3 IO Credits per second per GB of volume size (Baseline Performance). Burst up to 3000 IOPS by depleting the bucket. All volumes get an initial 5.4 million IO credits. 30 minutes at 3000 IOPS. Great for boots and initial workloads.

For volumes larger than 1000 GB Baseline is above burst. Credit system isn't used and you always achieve Baseline. Up to maximum for GP2 of 16000 IO Credits per second (Baseline Performance). Maximum throughput is 250 MiB/s.

GP2 is great for boot volumes, low-latency interactive apps, dev and test environments.

### GP3

Every GP3 volume starts with 3000 IOPS and 125 MiB/s (standard). Base price is 20% cheaper than GP2. Extra cost for up to 16000 IOPS and 1000 MiB/s.

## EBS Volume Types - Provisioned IOPS

IOPS can be adjusted independently of size. Designed for high performance situations (consistent low latency).

### IO1/IO2

Up to 64,000 IOPS per volume (4 times more than GP2/GP3). Up to 1000 MB/s throughput. Volume size range is from 4 GB to 16 TB. Maximum performance/size ratio: 50 IOPS/GB (IO1), 500 IOPS/GB (IO2). Per-instance performance cap for IO1: 260,000 IOPS and 7,500 MB/s; for IO2: 160,000 IOPS and 4,750 MB/s.

### IO2 Block Express

Up to 256,000 IOPS per volume. Up to 4000 MB/s throughput. Volume size range is from 4 GB to 64 TB. Maximum performance/size ratio: 1000 IOPS/GB. Per-instance performance cap: 260,000 IOPS and 7,500 MB/s (like IO1).

## EBS Volume Types - HDD-Based

For HDD, IO is measured as 1 MB block.

### ST1

Throughput-optimized. Cheaper than SSD volumes. Good for sequentially accessed data. Size range: from 125 GB to 16 TB. Maximum performance: 500 IOPS (500 MB/s). Baseline performance is 40 MB/s per 1 TB. Burst performance is 250 MB/s per 1 TB. Designed for frequently accessed throughput-intensive sequential data. Examples: big data, data warehouses, log processing.

### SC1

Cold HDD. Cheaper than ST1 (cheapest EBS volume type). Designed for infrequent workloads. Maximum performance: 250 IOPS (250 MB/s). Baseline performance: 12 MB/s per 1 TB. Burst performance: 80 MB/s per 1 TB. Size range: from 125 GB to 16 TB.

## Instance Store Volumes - Architecture

Block Storage Devices (like EBS, but local). Physically connected to one EC2 host; instances on that host can access volumes on the device. Highest storage performance in AWS (much higher than EBS). Included in the instance price. Attached **only at launch**. If an instance moves between hosts, then any data present on the instance store volume is lost (ephemeral host volumes).

Throughput numbers for some of the instance store types:
- D3 (HDD): 4.6 GB/s
- I3 (SSD): 16 GB/s (sequential throughput)

In general, instance store volumes can provide more IOPS and throughput than EBS.

An instance store provides temporary block-level storage for your instance. This storage is located on disks that are physically attached to the host computer. Instance store is ideal for temporary storage of information that changes frequently, such as buffers, caches, scratch data, and other temporary content, or for data that is replicated across a fleet of instances, such as a load-balanced pool of web servers.

An instance store consists of one or more instance store volumes exposed as block devices. The size of an instance store as well as the number of devices available varies by instance type.

The virtual devices for instance store volumes are ephemeral[0-23]. Instance types that support one instance store volume have ephemeral0. Instance types that support two instance store volumes have ephemeral0 and ephemeral1, and so on.

Some important notes on instance store volumes:
- local on EC2 Host
- added at launch only
- lost on instance move, resize or hardware failure
- high performance
- included in instance price
- temporary

## Choosing between the EC2 Instance Store and EBS

Choose EBS (avoid Instance Store) if you need:
- persistent storage
- resilient storage
- storage isolated from an instance lifecycle

50/50 cases:
- resilience with app in-build replication
- high performance

Choose Instance Store when:
- you have super high performance needs
- you want to cut costs (Instance Store often included in instance price)

If you need to use EBS and the cost is the highest priority, then use **ST1** or **SC1**. For throughput or streaming – **ST1**. But both **ST1** and **SC1** are not suitable for booting.

GP2/GP3 can deliver up to 16,000 IOPS. IO1/IO2 can deliver up to 64,000 IOPS (256,000 IOPS for IO2 Block Express). RAID0 set from EBS volumes can deliver up to 260,000 IOPS (instance limit). If you need more than 260,000 IOPS – use Instance Store.

## Snapshots, Restore & Fast Snapshot Restore (FSR)

EBS Snapshots are backups of data consumed within EBS Volumes – stored on S3.

Snapshots are incremental, the first being a full backup – and any future snapshots being incremental.

Snapshots can be used to migrate data to different availability zones in a region, or to different regions of AWS.

Volume restoration from a snapshot is a lazy gradual process. You can force a read of all data by tools like `dd`. Or you can enable the Fast Snapshot Restore (FSR), which will allow the immediate restore, but limited to 50 snapshots per region.

Snapshots are billed using a gigabyte/month metric (only for **used data**, not allocated data).

## Some useful volume-related Linux commands

```sh
lsblk # list block devices
sudo file -s /dev/xvdf # check the filesystem on the /dev/xvdf device
sudo mkfs -t xfs /dev/xvdf # create the xfs filesystem on the /dev/xvdf device
sudo mount /dev/xvdf /volumef # mount the /dev/xvdf device as /volumef folder
df -k # get info on disk devices

# to auto-mount the device as /volumef:
sudo nano /etc/fstab # add the line "UUID=<uuid>  /volumef  xfs  defaults,nofail", save and exit
sudo mount -a
```

## EBS Encryption

By default, no encryption is applied.

Accounts can be set to encrypt EBS volumes by default (with default KMS key). KMS key used to generate a per-volume unique DEK. Snapshots and future volumes (from those snapshots) use the same DEK. There is no way to remove the encryption from the volume. OS isn't aware of encryption (also, no performance loss). An unencrypted DEK stored only in EC2 instance memory.

## Network Interfaces, Instance IPs and DNS

EC2 instance always starts with one network interface: primary ENI (elastic network interface). Optionally, you can attach one or more secondary ENIs (they can also be in different subnets). EC2 instance and all its ENIs must be in the same AZ.

ENIs have:
- MAC address
- primary IPv4 private IP
  - plus internal DNS name, for example `ip-10-16-0-10.ec2.internal` for private IP `10.16.0.10`
- 0 or more secondary IPs
- 0 or 1 public IPv4
  - dynamic; will be de-allocated and re-allocated upon stopping and starting an instance
  - plus public DNS name, for example `ec2-3-89-7-136.compute-1.amazonaws.com` for public IP `3.89.7.136`
    - inside the VPC this name resolves to the primary IPv4 private IP
    - outside the VPC this name resolves to the public IPv4
- 1 Elastic IP per private IPv4
  - associated with the private IP either on primary or on secondary ENI
  - when associated with primary ENI, non-elastic public IPv4 is replaced with this Elastic IP
- 0 or more IPv6 addresses
- security groups
- source/destination check

You can detach secondary ENIs and move them to different instances.

### Tips on ENIs

- secondary ENI + MAC = Licensing
  - you can move secondary ENI to other instance along with licensing
- multi-homed (subnets) management & data
- different security groups – multiple interfaces
- OS doesn't see public IPv4

## Amazon Machine Images (AMI)

Can be used to launch an EC2 instance. AWS, Community or Marketplace provided. AMI are regional, they have a unique ID. AMI controls permissions: public, your account, specific accounts. You can create AMI from an EC2 instance.

AMI lifecycle:
1. Launch
2. Configure
3. Create image
4. Launch

When creating an AMI, AWS also creates snapshots of instance volumes. These snapshots (and corresponding device IDs) are referenced in Block Device Mapping. AMI is a container, which references snapshots.

### AMI tips

- specific AMI belong to one region and only works in that region
- AMI backing is the process of creating an AMI from a configured instance
- AMIs cannot be edited
- AMIs can be copied across regions
- by default created AMI is only accessible within your account

## EC2 Purchase Options (Launch Types)

### On-Demand

On-Demand instances are isolated, but multiple customer instances run on shared hardware. Instances of different sizes run on the same EC2 hosts (consuming a defined allocation of resources).

Per-second billing while an instance is running. Associated resources (such as storage) consume capacity, so bill, regardless of instance state.

Some On-Demand characteristics:
- default purchase option
- no interruption
- no capacity reservation
- predictable pricing
- no upfront cost
- no discount

Good for:
- short term workloads
- unknown workloads
- apps which can't be interrupted

### Spot

Spot pricing is AWS selling unused EC2 host capacity for up to 90% discount. The spot price is based on the spare capacity at a given time. If the spot price goes above your specified maximum price, your spot instances are terminated. Spot instances are not reliable. Never use Spot purchase for workloads which can't tolerate interruptions.

Good for:
- non time-critical
- anything which can be re-run
- bursty capacity needs
- cost sensitive workloads
- anything which is stateless

### Reserved

Reservation is a commitment made to AWS for the long-term consumption of EC2 resources. If reservation matches the instance, it would apply to that instance, either reducing or removing the per-second price. Unused reservation is still billed. Reservations can be purchased for a particular type of instance and locked to an AZ specifically, or to a region. If you lock a reservation to an AZ, you can only benefit when launching instances into that AZ, but it also reserves capacity. If you purchase a reservation for a region, it doesn't reserve capacity, but can benefit any instances which are launched into any AZ of that region.

Choices:
- the term: commit for **one year** or for **three years** (greater discount)
- payment structure: **no upfront** (reduces per-second fee, least discount), **partial upfront** (partial advance plus reduced per-second fee, "middle ground") or **all upfront** (all cost in advance, greatest discount)

Good for:
- known usage
- consistent access to compute
- long-term requirements

Variations:
- Standard Reserved
- Scheduled Reserved
  - commitment is: frequency, duration, time
  - ideal for long-term usage which doesn't run constantly; examples:
    - batch processing daily for 5 hours starting at 23:00
    - weekly data, sales analysis every friday for 24 hours
    - 100 hours of EC2 capacity per month
  - doesn't support all instance types or regions
  - minimum:
    - 1,200 hours per year
    - 1 year term

### Dedicated Hosts

EC2 host, which is allocated to you fully. You pay for a host itself, which is designed for a specific family of instances. Any instances which run on that host have no per-second charge. Have a **host affinity** feature which links instances to hosts.

Reasons to use:
- you have a software with license based on sockets or cores in a physical machine

## Dedicated Instances

Your instances run on an EC2 host and no other customers use the same hardware. You don't own or share the host. Extra charges for instances, but dedicated hardware.

Extra fees:
- one-off hourly fee for any regions where you use Dedicated Instances
- fee for the Dedicated Instances themselves

Used when:
- you have strict requirements to not share infrastructure

### Capacity Reservations

Priority order:
1. Reserved
2. On-Demand
3. Spot

Capacity reservation is different from reserved instance purchase. Two different components: billing and capacity, which can be used in combination or individually.

Regional reservation provides a billing discount for valid instances launched in any AZ in that region. While flexible, they don't reserve capacity within an AZ, which is risky during major faults when capacity can be limited.

Zonal reservation only apply to one AZ providing billing discounts and capacity reservation in that AZ.

For regional or zonal reservations you still commit to 1 or 3 years.

On-Demand capacity reservation can be booked to ensure you always have access to capacity in an AZ when you need it, but at full on-demand price. No term limits, but you pay regardless of if you consume it.

### EC2 Savings Plan

An hourly commitment for a 1 or 3 year term. A reservation of general compute $ amounts ($20 per hour for 3 years; up to 66% cost reduction). Or a specific EC2 savings plan: flexibility on size & OS (up to 77% cost reduction). Supported compute products: EC2, Fargate, Lambda. Products have an on-demand rate and a savings plan rate. Resource usage consumes savings plan commitment at the reduced savings plan rate. Beyond your commitment On-Demand rate is used.

## Instance Status Checks & Auto Recovery

Every EC2 instance have 2 high-level per-instance checks. Each of them represents a separate set of tests. First status check is **system status**. The second is **instance status**. Failure of the **system status check** can indicate:
- loss of system power
- loss of network connectivity
- host software issues
- host hardware issues

Failure of the **instance status check** can indicate:
- corrupted file system
- incorrect instance networking
- OS kernel issues

Ways to handle failed status checks:
- manually
  - stop and start an instance
  - restart an instance
  - terminate and recreate an instance
- automatically – ask EC2 to perform an action on a failed instance (enabled via status check alarm):
  - stop
  - reboot
  - terminate
  - perform an auto-recovery: move the instance to a new host (within the same AZ) and start it with the same configuration
    - doesn't work if the whole AZ fails
    - doesn't work for older types of instances
    - doesn't work for instances with instance store volumes

You can enable instance termination protection in instance settings: protects an instance from accidental manual termination and requires an extra permission to disable.

You can change the shutdown behavior in instance settings: by default an instance is stopped, but it also can be terminated.

## Horizontal & Vertical Scaling

### Vertical Scaling

Increasing the instance's capacity by upgrading to a more advanced instance.

Cons:
- Each resize requires a reboot (disruption, downtime).
- Larger instances often carry a price premium.
- There is an upper cap on performance: instance size.

Pros:
- No application modification required.
- Works for all applications, even monoliths.

### Horizontal scaling

Just add more instances. N copies of running application. Instances sit behind load balancer, which distributes the load from users between the instances.

User sessions are very important for horizontal scaling.

Cons:
- Requires application support or off-host sessions (stateless instances).

Pros:
- No disruption when scaling.
- No limits.
- Often less expensive.
- More granular.

## Instance Metadata

EC2 service which provides data to instances. Accessible inside all instances: `http://169.254.169.254/latest/meta-data`. Allows to query the data about an instance' environment (hostname, events, security group, etc), networking, authentication, user data. The metadata service is not authenticated and not encrypted, so it is exposed to everyone who has SSH access to the instance.

Examples:
- `curl http://169.254.169.254/latest/meta-data/public-ipv4` to get instance' public IPv4 address (inaccessible within the OS)
- `curl http://169.254.169.254/latest/meta-data/public-hostname` to get instance' public hostname (inaccessible within the OS)

To use the EC2 metadata query tool:
```sh
wget http://s3.amazonaws.com/ec2metadata/ec2-metadata
chmod u+x ec2-metadata

ec2-metadata --help
ec2-metadata -a # AMI ID
ec2-metadata -z # AZ
ec2-metadata -s # Security groups
```

## Quiz

- What are the three main states of EC2 instances (choose all which apply)
  - RUNNING
  - STOPPED
  - TERMINATED
- What is true of instance store volumes
  - They are temporary (ephemeral) storage
  - Data stored on them can be lost when an EC2 instance stops and starts
  - Data stored on them can be lost if a hardware failure occurs
- If the AZ which an EC2 instance is running in fails, what happens to the instance?
  - The instance will remain failed until at least when the AZ recovers
- Can an EC2 instance be migrated between AZs?
  - No - but an AMI can be created from an instance and used to provision a clone in another AZ
- What kind of use-case suits using IO1 EBS volumes
  - When maximum consistent IOPS is a priority and data is important
- If you need to be able to specify performance requirements (IOPS) independent of volume size which volume type should you choose?
  - IO1
- How many instances can a GP2 volume be attached to at the same time
  - 1
- Can EBS volumes be attached to instances in ANY AZ?
  - No, only instances in the same AZ as the volume
- When should instance store volumes be used? (choose all that apply)
  - For replaceable data
  - For temporary data
  - For max IO
- If you have a short term workload which needs the cheapest EC2 pricing but cant tolerate interruption which billing model should you pick
  - On-demand