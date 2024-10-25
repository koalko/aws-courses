# Relational Database Service (RDS)

## ACID and BASE

Both ACID and BASE are DB transaction models.

**CAP Theorem**: choose 2 of:

- Consistency
- Availability
- Partition Tolerance

ACID focuses on consistency; BASE focuses on availability.

ACID means that transactions are:

- **Atomic**: transaction operations are either all successful, or all fail
- **Consistent**: transactions move the database from one valid state to another (nothing in-between is allowed)
- **Isolated**: if multiple transactions occur at once, they don't interfere with each other (each executes as if it's the only one)
- **Durable**: once committed, transactions are durable (stored on non-volatile memory, resilient to power outages or crashes)

ACID limits DB scaling.

BASE means:

- **Basically Available**: read/write ops are available as much as possible, but without any consistency guarantees
- **Soft state**: the database doesn't enforce consistency; it is offloaded onto the application/user
- **Eventually consistent**: if we wait long enough, reads from the system will be consistent

## Hosting a database on EC2 instance

Generally considered a bad practice:

- admin overhead (managing EC2 and DBHost)
- backups / disaster recovery management
- EC2 is single AZ
- features some of AWS DB products provide
- no serverless, no easy scaling
- replication (skills, setup time, monitoring & effectiveness)
- performance (AWS invest time into optimization)

Reasons to do it:

- access to DB instance OS
- advanced DB options tuning (`DBROOT`)
- vendor demands
- DB or DB version AWS don't provide
- architecture AWS don't provide (replication/resilience)

## Relational Database Service (RDS) Architecture

Commonly referred to as _database as a service_ (DBaaS), but it is rather _database server as a service_ (DBSaaS). Managed service: no access to OS, no SSH access (exception: RDS Custom). Works inside VPC. Supports multiple databases on one DB server (instance). Supports multiple DB engines:

- MySQL
- MariaDB
- PostgreSQL
- Oracle
- Microsoft SQL Server

It is important to note that Amazon Aurora is a different product.

RDS Subnet Group allows to specify subnets to use by RDS (can be located in different AZs). When creating RDS instance with high availability, RDS will pick one AZ for the **primary** instance and another one for **standby**. Primary instance being replicated to standby instance via the _synchronous replication_. Read replicas use _asynchronous replication_.

RDS can be configured with public addressing allowing access from the public internet, but this is generally discouraged.

Each RDS instance can have multiple databases, located on a dedicated per-instance EBS storage.

RDS backups and snapshots stored in S3.

With RDS you are billed by the allocated resources:

- instance size and type
- multi AZ (if enabled)
- storage type and amount
- data transferred
- backups and snapshots
- licensing (if applicable)

## Multi AZ

### Multi AZ Instance

In a Multi AZ setup there is a primary instance (used for regular DB operations) and a standby instance (synchronously replicated from primary), which will be used instead of primary in case of an outage (by switching IP under the instance hostname). There is no free tier for Multi AZ setup (extra cost for a replica). There can only be one standby replica, and it cannot be used for reads/writes. Failover will take 60-120 seconds. Multi AZ only supports different AZs within a single region. Backups are taken from standby replica to improve performance.

Failover reasons might be:

- AZ outage
- primary failure
- manual failover
- instance type change
- software patching

### Multi AZ Cluster

One **writer** instance is synchronously replicated to two or more **reader** instances. Data is committed when 1+ reader finishes writing. Cluster endpoint points at the writer and can be used for reads, writes and administration. Reader endpoint directs any reads to an available reader instance. Instance endpoints point at specific instance. Generally they are used for testing / fault finding.

Much faster hardware (Graviton + MVME SSD Storage). Fast writes to local storage, flushed to EBS.

Replication is done via transaction logs (more efficient). Failover is faster (~35s + transaction log apply).

## RDS Automatic Backup, Snapshots and Restore

Both backups and snapshots use AWS managed S3 buckets (not accessible from AWS console). S3 replicates data across AZs, so backups and snapshots are regionally resilient.

Snapshots are not automatic (manually initiated) and taken from the instance (for all the DBs within the instance). First snapshot is full; following snapshots are incremental. Snapshots live past the RDS instance and need to be removed manually.

Backups are similar to snapshots, but automated (once per day). Basically, backups are automated snapshots. They are removed automatically; you can set retention period from 0 to 35 days.

Additionally, every 5 minutes DB transaction logs are stored to S3.

RDS can also replicate backups (both snapshots and transaction logs) to another region. Charges apply to cross-region data copy and the storage in the destination region. This is not a default behavior; it needs to be enabled within the automated backups.

RDS Restore creates a new RDS instance with a new address. With snapshots, you restore DB at a single point in time (snapshot creation time). With backups, you can choose any 5 minute point in time (using snapshot + transaction logs). Restores are not fast.

## RDS Read-Replicas

Read replicas aren't part of a primary instance (needs to be supported by the application). Read replicas get data from a primary instance via asynchronous replication. Read replicas can even be created in another region (cross-region replication).

You can create up to 5 read replicas per RDS DB instance, each providing an additional instance of read performance.

Read replicas can have own read replicas, but in that case lag might be a problem.

Snapshots and backups improve RPO (recovery point objective), but doesn't solve RTO (recovery time objective). Read replicas offer near 0 RPO and low RTO (they can be quickly promoted). Read replicas should be used for disaster recovery only in case of a failure, not in case of data corruption (it will be replicated, and replicas will be corrupted as well). Read replicas are read-only until they are promoted.

## RDS Data Security

### Encryption

SSL/TLS (encryption in transit) is available for RDS, can be mandatory.

RDS supports EBS volume encryption (KMS), which is handled by the Host/EBS. AWS or customer managed CMK generates data keys, which are used for data encryption operations. Storage, logs, snapshots and replicas are also encrypted. Encryption cannot be removed once activated.

RDS MSSQL and RDS Oracle support Transparent Data Encryption (TDE), which is handled within the DB engine. RDS Oracle also supports integration with CloudHSM (much stronger key controls).

### IAM Authentication

RDS local DB account configured to use AWS Authentication Token. Policy attached to user or role maps that IAM identity onto the local RDS user. This allows IAM identity to run `generate-db-auth-token`, which creates a token with 15 minutes validity, which can be used in place of a DB user password. This process only covers the authentication; authorization is covered by internal DB mechanisms.

## RDS Custom

Fills the gap between RDS and EC2 running a DB engine. RDS is fully managed, so OS/engine access is limited. DB on EC2 is self-managed, but has overhead.

RDS Custom currently works for MS SQL and Oracle. Can connect using SSH, RDP, Session Manager.

With regular RDS the database infrastructure is running in AWS managed environment. With RDS Custom you can see all the DB infrastructure resources (EC2 instances, EBS volumes, S3 backups) in your AWS account.

Access RDS Custom Database Automation, pause it to perform DB customization and resume it afterwards.

## Aurora Architecture

Very different from RDS. A cluster, which consists of single primary instance + zero or more replicas. Replicas can be used for reads (providing read scaling + availability). No local storage; uses cluster volume, hence faster provisioning, improved availability and performance. Cluster volume is SSD-based (high IOPS, low latency), has 6 storage nodes across different AZs and supports up to 128 TiB volume. All of the instances have access to these storage nodes. Replication happens at the storage level, so no resources are consumed on the instances or the replicas during the replication process. Only primary instance can write to the storage; all the instances can read from it. Aurora supports up to 15 replicas. Replicas can be added and removed without requiring storage provisioning.

Recovery from disk failures is fully automated with Aurora.

Storage is billed based on what's used; by "high water mark" (billed for the most used; might be disabled in more recent Aurora versions). Storage which is freed up can be re-used.

With Aurora you have cluster endpoint (always points at the primary instance; can be used for reads/writes) and reader endpoint (will load balance reads across all the available replicas).

Aurora doesn't have free tier (doesn't support micro instances). Beyond RDS single AZ (micro) Aurora offers better value (?). Compute: hourly charge, per second, 10 minute minimum. Storage: GB/month consumed, IO cost per request. 100% of DB size in backups are included.

Backups work in the same way as RDS. Restores create a new cluster. Backtrack can be used which allow in-place rewinds to a previous point in time. Fast clones make a new database much faster than copying all the data (copy-on-write).

## Aurora Serverless

Aurora Serverless to Aurora is that Fargate is to ECS.

ACU – Aurora Capacity Units – represent the certain amount of compute and memory. Aurora Serverless cluster has a min and max ACU. Cluster adjusts based on load. Can go to zero and be paused. Consumption billing per-second basis. Same resilience as Aurora (6 copies across AZs).

Use cases:

- infrequently used applications
- variable workloads
- unpredictable workloads
- development and test databases
- multi-tenant applications

## Aurora Global Database

Global-level replication using Aurora from a master region to up to 5 secondary regions (up to 16 read replicas in each of secondary regions; ~1s replication time). Great for cross-region disaster recovery (DR) and business continuity (BC). Global Read Scaling: low latency performance improvements. Replication occurs at a storage layer, so no impact on DB performance.

## Multi-master writes

Default Aurora mode is single-master: one r/w and 0+ read-only replicas. Cluster endpoint is used for write, read endpoint is used for load-balanced reads. Failover takes time: replica promoted to r/w.

In multi-master mode all instances are r/w. There is no load-balanced endpoint; application needs to choose the instance to connect to. When one instance receives a write request, it immediately propose this data to be committed to all of the storage nodes in the cluster, and when waits for a "quorum" to mark write as committed and also pass this write to other instances so they able to update their cache.

## RDS Proxy

Reasons to use:

- opening and closing connections is not free
  - consume resources
  - takes time (hence latency)
  - especially noticeable with serverless
- handling failure of DB instance is hard

DB proxies help, but managing them is not trivial (scaling/resilience).

Instead of connecting to a DB directly, application connects to a proxy, which maintains a pool of connections to a DB.

RDS Proxy is a managed proxy, which runs from a VPC.

When to use RDS Proxy:

- _too many connections_ errors
- when using T2/T3 (smaller/burst) instances for DB
- AWS Lambda (time saved / connection reuse / IAM Auth)
- when DB connection latency matters
- when resilience to DB failure is a priority (RDS proxy can reduce the time for failover and make it transparent to an application)

Key facts:

- Fully managed DB proxy for RDS/Aurora
- Auto scaling, highly available by default
- Provides connection pooling (reduces DB load)
- Only accessible from VPC
- Accessed via proxy endpoint
- Can enforce SSL/TLS
- Can reduce failover time by over 60%
- Abstracts failure away from an application

## Database Migration Service (DMS)

A managed database migration service, which runs using a replication instance. Source and destination endpoints point at source and destination databases. At least one of the endpoints must be running on AWS. Supported database engines:

- MySQL
- Aurora
- MSSQL
- MariaDB
- MongoDB
- PostgreSQL
- Oracle
- Azure SQL
- ...

On a replication instance you define replication tasks. Jobs can be:

- full load (one-off migration of all data)
- full load + CDC (change data capture) for ongoing replication which capture changes
- CDC only (if you want to use an alternative method to transfer the bulk DB data, such as native tooling)

Schema conversion tool (SCT) can assist with schema conversion. It is a standalone app which is used when converting from one database engine to another (including DB → S3). SCT is not used when migrating between DBs of the same type.

Larger migrations might be multi-TB in size. Moving data over networks take time and consumes capacity. DMS can utilize Snowball:

1. Use SCT to extract data locally and move to a Snowball device.
2. Ship the device back to AWS. They load data onto an S3 bucket.
3. DMS migrates from S3 into the target store.
4. CDC can capture changes, and via S3 intermediary they are also written to the target database.

## Quiz

- RDS is designed for what database model
  - SQL
- What feature of RDS allows the system to scale for READS
  - RDS RR
- Which feature of RDS provides HA Functionality
  - RDS MultiAZ
- How is the standby node of an RDS MultiAZ accessed?
  - It's only accessed after a failover when it becomes the primary instance
- Which AWS Product can be used to help move data TO or FROM AWS in a controlled and configurable way
  - DMS
- What type of backup should you take of RDS if you want it to be available for up to 1 year
  - Manual Snapshot
- When restoring RDS from a snapshot or backup is any application reconfiguration required?
  - A different endpoint address is created when restoring ... authentication remains the same
- Which Managed SQL database product in AWS supports 3+ AZ Resilience
  - Aurora
- Which AWS Managed SQL Database product horizontally scales and can reduce to 0 and pause when there is no load
  - Aurora Serverless
- What RDS DB product supports HA where all instances can be used for writes
  - Aurora (Multi-master)