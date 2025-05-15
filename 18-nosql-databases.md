# NOSQL Databases & DynamoDB

## DynamoDB - Architecture

DynamoDB is a NoSQL public database-as-a-service (DBaaS) product:

- Capable of handling simple key/value data or data with a structure (document).
- No self-managed servers or infrastructure required.
- Supports a range of scaling options: manual/automatic provisioned performance in/out or on-demand.
- Highly resilient across AZs and optionally global.
- Really fast: single-digit milliseconds (SSD based).
- Handles backups, allows for point-in-time recovery, provides encryption at rest.
- Supports event-driven integration: generate events and configure actions when data within DynamoDB changes.

A table is a grouping of items with the same primary key. A table can have an infinite number of items within it. Primary key types:

- simple: partition (PK)
- composite: partition (PK) & sort (SK)

Every item in the table has to use the same primary key, and it has to have unique value for that primary key. Other than that, there are no restrictions on item's attributes: DynamoDB has no rigid attribute schema.

Maximum item size: 400KB.

You can provision a table with provisioned capacity or on-demand capacity. In DynamoDB capacity means speed (performance). Capacity is set on a table:

- Writes: 1 WCU = 1KB per second
- Reads: 1 RCU = 4KB per second

Backup types:

- on-demand
  - full copy of table
  - retained until removed
  - restore:
    - same or cross-region
    - with or without indexes
    - adjust encryption settings
- point-in-time recovery (`PITR`)
  - not enabled by default
  - enabled per-table
  - continuous record of changes allows replay to any point in the window
  - 35 day recovery window
  - restore: restore from the recovery window into a new table with 1 second granularity

DynamoDB considerations:

- (exam) DynamoDB should be the preference for:
  - NoSQL
  - key/value
- (exam) DynamoDB is not suited for relation data
- accessed via console, CLI, API
  - SQL is not supported
- billed on RCU, WCU, storage and features

## DynamoDB - Operations, Consistency and Performance

### Capacity units

Every operation consumes at least 1 RCU/WCU.

1 RCU is one 4KB read operation per second (100B item read still consumes 1 RCU; 400KB item read consumes 100 RCUs). 1 WCU is one 1KB write operation per second. Every table has a RCU and WCU burst pool (300 seconds).

### Capacity modes

DynamoDB allow you to pick one of two capacity modes when creating a table:

- on-demand
  - unknown
  - unpredictable
  - low admin
  - price per million R or W units
  - price is up to 5 times more than provisioned
- provisioned
  - RCU and WCU set on a per table basis

### Query

Query operation is one way you can retrieve data from DynamoDB table. Query accepts a single `PK` value and optionally a `SK` or `range`. Capacity consumed is the size of all returned items. Further filtering discards data, but capacity is still consumed. Can only query on `PK` or `PK+SK`.

### Scan

Scan is the least efficient operation for fetching data from DynamoDB, but it is also the most flexible. Scan moves through a table consuming the capacity of every item. You have complete control on what data is selected, any attributes can be used and any filters applied but scan consumes capacity for every item scanned through.

### Consistency Model

DynamoDB consistency modes:

- eventually consistent
- strongly (immediately) consistent

DynamoDB replicates its data over multiple AZs; each point called `Storage Node`. One of storage nodes elected as a leader.

Example:

1. Bob updates a DynamoDB item, removing an attribute.
2. DynamoDB directs the write at the leader storage node.
3. The Leader node replicates data to other nodes, typically finishing within a few milliseconds; let's assume one of three nodes not yet updated.
4. Eventually consistent reads check 1/3 nodes: could be unlucky with stale data if a node is checked before replication completes. 50% of the cost vs strongly consistent.
5. Strongly consistent reads connect to the leader node to get the most up-to-date copy of data.

Not every application or access type tolerates eventual consistency. Select the model as appropriate.

### WCU Calculation

For example, you need to store 10 items per second, 2.5K average size per item. To calculate WCU per item, round up `item size / 1KB` (in this example: `3`). Next, multiply that value by average number per second (in this example: `3 * 10 = 30` WCU required).

### RCU Calculation

For example, you need to retrieve 10 items per second, 2.5K average size. To calculate RCU per item, round up `item size / 4KB` (in this example: `1`). Next, multiply that value by average read ops per second (in this example: `1 * 10 = 10` strongly consistent RCU required, or `10 / 2 = 5` eventually consistent RCU required – for 50% cost).

## DynamoDB - Local (`LSI`) and Global (`GSI`) Secondary Indexes

Query is the most efficient operation in DDB. Query can only work on 1 `PK` value at a time; and optionally a single, or range of `SK` values. Indexes are alternative views on table data. `LSI` allow you to create a view using different `SK`; `GSI` allow you to create a view using different `PK` and `SK`. Both `LSI` and `GSI` support projection (some or all attributes).

Indexes are sparse: only items, which have a value in the index alternative sort key are added to the index.

### `LSI`

- **Must** be created with a table
- Limit: 5 `LSI` per base table
- Allow for alternative `SK` on the table
- Shares the RCU and WCU with the table
- Attributes: `ALL`, `KEYS_ONLY`, `INCLUDE`

### `GSI`

- Can be created **at any time**
- Default limit of 20 per base table
- Alternative `PK` and `SK`
- Have their own RCU and WCU allocations
- Attributes: `ALL`, `KEYS_ONLY`, `INCLUDE`
- Always eventually consistent; replication between base and `GSI` is asynchronous

### `LSI` and `GSI` Considerations

- Careful with projection (`KEYS_ONLY`, `INCLUDE`, `ALL`)
- Queries on non-projected attributes are expensive
- Use `GSI`s as default, `LSI` only when strong consistency is required
- Use indexes for alternative access patterns

## DynamoDB - Streams & Lambda Triggers

### Streams

DynamoDB stream is a time ordered list (24-hour rolling window) of item changes in a table. Uses Kinesis Streams under the hood. Enabled on a per table basis. Records inserts, updates and deletes. Different view types influence what is in the stream.

Stream view types:

- `KEYS_ONLY`: only `PK` and optionally `SK`
- `NEW_IMAGE`: entire item data after the change
- `OLD_IMAGE`: entire item data before the change
- `NEW_AND_OLD_IMAGES`: entire item data before and after the change

### Triggers

Item change generate an event. That event contains the data which changed. When event is generated, an action is taken (via Streams + Lambda) using that data. Used for reporting, analytics, data aggregation, messaging or notifications.

Steps:

1. Item change occurs in a table with streams enabled.
2. Stream record is added onto stream.
3. Lambda function is invoked when stream event occurs. Function is passed the **view** data as an **event**.

## DynamoDB - Global Tables

Global tables provides multi-master cross-region replication. Tables are created in multiple regions and added to the same global table (becoming replica tables). `Last writer wins` is used for conflict resolution. Reads and Writes can occur to any region. Generally sub-second replication between regions. Strongly consistent reads **only** in the same region as writes. Cross-region writes/reads are eventually consistent. Provides global HA and global DR/BC (disaster recovery/business continuity).

## DynamoDB - Accelerator (DAX)

DAX is an in-memory cache for DynamoDB.

### Traditional cache vs DAX

With traditional cache:

1. Application checks cache for data; a `cache miss` occurs if data isn't cached.
2. Data is loaded from the database with a separate operation and SDK.
3. Cache is updated with retrieved data. Subsequent queries will load data from cache as a `cache hit`.

With DAX:

0. DAX SDK is pre-installed on the application side, which makes DAX and DynamoDB the same from the application perspective.
1. Application uses the DAX SDK and makes a single call for the data which is returned by DAX.
2. DAX either returns the data from its cache or retrieves it from the database and then caches it.

With DAX there is less complexity for the app developer (tighter integration).

### DAX Architecture

DAX operates from within the VPC and is designed to be deployed to multiple AZs. DAX has a primary node in one AZ and replica nodes in others. DAX cluster interacts with application + DAX SDK and with DynamoDB.

Two types of DAX caches:

- `item cache`: holds results of `(Batch)GetItem`
- `query cache`: holds data based on query/scan parameters

DAX is accessed via endpoint. Cache **hits** are returned in microseconds, **misses** – in milliseconds. Write-through is supported: data is written to DAX & DDB simultaneously. If a cache **miss** occurs, data is also written to the primary node of the cluster (and replicated to replica nodes).

### DAX Considerations

- primary node (write ops) and replicas (read ops)
- nodes are HA; primary failure leads to election
- in-memory cache: much faster reads, reduced costs
- scale up (bigger instances) and scale out (additional instances)
- supports write-through
- deployed within a VPC
- DAX is useful if:
  - you have read-heavy or bursty workloads (to get better performance and lower cost)
  - you need very low response times
- DAX might not be a good fit if:
  - application require strongly consistent reads
  - you don't really need low response times
  - application is write-heavy and uses read operations infrequently

## DynamoDB - TTL

TTL allow you to set timestamp for automatic items deletion. When TTL is enabled on a table, a specific attribute is selected for TTL (contains UNIX timestamp in seconds).

A per-partition process periodically runs, checking the current time (in seconds since epoch) against the value in the TTL attribute. Items where the TTL attribute is older than the current time are set to expired.

Another per-partition background process scans for expired items and removes them from tables and indexes; also, a delete is added to streams if enabled.

Any delete operations, caused by TTL, are background system processes; they don't impact table performance and aren't chargeable.

You can also configure a stream on TTL processes (24-hour window).

## Amazon Athena

Serverless interactive querying service. You can take data, stored within S3, and perform ad-hoc queries on that data, paying for only the amount of data consumed while running the query (plus the S3 storage). No base monthly cost, no per-minute or per-hour charges.

Athena uses `schema-on-read`: a lens, through which you see the data in a certain way, while the original data is unchanged. Schema translates data to relational-like when read. Output can be sent to other services.

Supports standard formats of structured, semi-structured and unstructured data. Some of supported data formats:

- XML
- JSON
- CSV/TSV
- AVRO
- PARQUET
- ORC
- Apache
- CloudTrail
- VPC Flowlogs

Athena is great for:

- queries where loading/transformation isn't desired
- occasional / ad-hoc queries on data in S3
- cost conscious serverless querying scenarios
- querying AWS logs: VPC Flow Logs, CloudTrail, ELB logs, cost reports, etc
- querying AWS Glue Data Catalog & Web Server Logs
- querying other data sources with `Athena Federated Query`

You need to setup an S3 bucket as a query result location.

### Example of setting up and querying an Athena DB

Create the Athena DB:

```sql
CREATE DATABASE A4L;
```

Create the Athena DB Table:

```sql
CREATE EXTERNAL TABLE planet (
  id BIGINT,
  type STRING,
  tags MAP<STRING,STRING>,
  lat DECIMAL(9,7),
  lon DECIMAL(10,7),
  nds ARRAY<STRUCT<ref: BIGINT>>,
  members ARRAY<STRUCT<type: STRING, ref: BIGINT, role: STRING>>,
  changeset BIGINT,
  timestamp TIMESTAMP,
  uid BIGINT,
  user STRING,
  version BIGINT
)
STORED AS ORCFILE
LOCATION 's3://osm-pds/planet/';
```

Locational query example:

```sql
SELECT * from planet
WHERE type = 'node'
  AND tags['amenity'] IN ('veterinary')
  AND lat BETWEEN -27.8 AND -27.3
  AND lon BETWEEN 152.2 AND 153.5;
```

## ElastiCache

In-memory database for applications with high performance requirements. Much faster than, for example, RDS database; but not persistent. Provides two managed engines as a service:

- Redis
- Memcached

Can be used to cache data – for **read heavy** workloads with **low latency** requirements. Can reduce database workloads, which improves performance and reduces costs.

Can be used to store session data for stateless servers.

Requires application code changes.

### Redis vs Memcached

- Memcached
  - Simple data structures
  - No replication
  - Multiple nodes (sharding)
  - No backups
  - Multi-threaded
- Redis
  - Advanced structures
  - Multi-AZ replication (scale reads)
  - Backup & restore
  - Transactions

## Redshift Architecture

Petabyte-scale data warehouse. Designed for reporting and analytics. OLAP (column-based), not OLTP (row/transaction). Pay as you use.

Direct query S3 using `Redshift Spectrum`. Direct query other DBs using `federated query`.

Integrates with AWS tooling such as `Quicksight`. Provides SQL-like interface for JDBC/ODBC connections.

Server-based (not serverless). Uses a cluster architecture; cluster is a private network – you can't access most of the cluster directly.

Redshift runs with multiple nodes and a high-speed networking between those nodes. Runs in **one AZ** in a VPC (not highly-available by design). Have a `leader node`, which manages communication with client programs and all communication with compute nodes, and develops execution plans to execute queries. `Compute nodes` runs the queries which are assigned by the `leader node` and store the data, loaded into the system. Compute node is partitioned into slices; each slice is allocated a portion of node's memory and disk space, where it processes a portion of a workload, assigned to that node. The leader node manages distributing data to the slices and a portions of a workload for any queries or other operations onto the slices. The slices work in parallel to complete the operation.

A node might have 2, 4, 16 or 32 slices – it depends on a resource capacity of that node.

Supports VPC security, IAM permissions, KMS at rest encryption, CW monitoring.

**(Might be important for the exam)**: Redshift `Enhanced VPC Routing` allow to route traffic based on VPC networking configuration: controlled by the security groups, NACLs, use custom DNS and VPC gateways.

Ways to import data to Redshift:

- load from S3
- copy from DynamoDB
- migrate via DMS
- stream from Firehose

### Redshift DR and Resilience

Automatic incremental backups occur every ~8 hours or every 5GB of data and by default have a 1-day retention (configurable up to 35 days). Manual snapshots can be taken at any time – and deleted by an admin as required.

Redshift can be configured to copy snapshots to another region for DR – with a separate configurable retention period.

## Quiz

- What is an RCU?
  - 4KB of read per second
- When can an LSI be created?
  - At table creation time
- When can a GSI be created?
  - At any time
- Which of the options below are qualities or benefits of an LSI
  - Shared Capacity Settings with the table
  - Alternative SK
- Which of the options below are qualities or benefits of an GSI
  - Its own capacity allocation
  - Alternative PK and SK
- What is true about DAX (choose 4)
  - Supports read caching of items and query/scan results
  - Supports write-through and read caching
  - Runs from within a VPC
  - Good for read heavy workloads
- What 4 view types are available with DDB Streams
  - KEYS_ONLY
  - NEW_IMAGE
  - OLD_IMAGE
  - NEW_AND_OLD_IMAGES
