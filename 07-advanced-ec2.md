# Advanced EC2

## Bootstrapping EC2 with User Data

EC2 Bootstrapping is the process of configuring an EC2 instance to perform automated install & configuration steps 'post launch' before an instance is brought into service.

With EC2 this is accomplished by passing a script via the User Data part (http://169.254.169.254/latest/user-data) of the Meta-data service – which is then executed by the EC2 instance OS (only once; at launch).

Bootstrapping allows EC2 Build Automation. User Data is opaque to EC2: it's just a block of data. It's not secure (shouldn't be used for passwords or other long-term credentials). It's limited to 16 Kb in size. Can be modified when instance stopped (and will be available inside the instance), but only executed once at launch.

The boot-time-to-service-time is usually minutes for AMI-to-instance, but post-launch-time (to configure the instance) can take up to hours. Bootstrapping is aimed to shorten this time. AMI baking also reduces post-launch time, but is less flexible. The optimal way is to combine AMI baking (for longer tasks, which are primarily static; like large software installation) and Bootstrapping (for shorter tasks, which are often subject to change; like applying configuration).

User Data is configured in `Advanced details` section when the EC2 instance is created/launched.

To see the User Data from the running instance:

```sh
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/meta-data/
```

## Enhanced Bootstrapping with CFN-INIT (CloudFormation::Init)

There is a `cfn-init` helper script installed on EC2 OS. It is basically a simple configuration management system. While User Data provides a procedural approach, `cfn-init` describes a desired state. Allows to install/configure/control:

- packages
- groups
- users
- sources
- files
- commands
- services

Provided with directives via `Metadata` and `AWS::CloudFormation::Init` on a CFN resource.

The `cfn-signal` script is used to report to CloudFormation the state of the `cfn-init` command. If CFN stack has `EC2Instance::CreationPolicy::ResourceSignal` option set, the stack will only move to `CREATE_COMPLETE` state if `cfn-signal` sent "SUCCESS".

## EC2 Instance Roles

IAM Role with necessary permissions allows EC2 Service to assume it. `Instance Profile` is a wrapper around an IAM Role, which allows permissions to "get inside" the instance. It is actually the Instance Profile being attached to EC2 Instance, not the IAM Role directly.

Temporary credentials (provided by the assumed role) delivered via instance meta-data (`iam/security-credentials/<role-name>`). Application, running inside the instance, can access these credentials, which are automatically rotated (always valid).

You should always use instance roles instead of adding long-term access keys directly to the instance. CLI tools will use role credentials automatically.

## AWS Systems Manager (SSM) Parameter Store

Regional public service. Storage for configuration and secrets.

Allows to store:

- String
- StringList
- SecureString

Supports hierarchy (for example, `/wordpress/DBUser`) and versioning. Supports encryption via KMS.

Has the concept of public parameters: for example, latest AMIs per region.

Changes in parameters can create events.

Example of AWS CLI commands to interact with Parameter Store:

```sh
aws ssm get-parameters --names /some/path/parameter
aws ssm get-parameters-by-path --path /some/path/
aws ssm get-parameters-by-path --path /some/path/ --with-decryption
```

## System and Application Logging on EC2

CloudWatch is used for metrics, CloudWatch Logs is used for logging, but neither natively capture the data inside an instance. CloudWatch Agent (plus configuration and permissions) is required to provide visibility of such data.

Steps required to enable the system/application logging from EC2 instance:

1. SSH into the instance and install CloudWatch Agent with `sudo dnf install amazon-cloudwatch-agent`.
2. Create a EC2 IAM Role with `CloudWatchAgentServerPolicy` and `AmazonSSMFullAccess` managed policies (for example, named `CloudWatchRole`) and attach this role to EC2 instance.
3. SSH into the instance and start CloudWatch Agent configuration wizard with `sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard`.
4. Apply the fix so CloudWatch Agent will be able to start:
   ```sh
   sudo mkdir -p /usr/share/collectd/
   sudo touch /usr/share/collectd/types.db
   ```
5. Start the CloudWatch Agent with `sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:<agent-config-ssm-parameter-name> -s`.

## EC2 Placement Groups

Types:

- `Cluster` – pack instances close together
  - same rack; sometimes same host
  - all members have direct connections to each other
  - one AZ only (locked when launching the first instance)
  - can span VPC peers, but this impacts performance
  - requires a supported instance type
  - better to use same type of instance (not mandatory)
  - better to launch all instances at the same time (not mandatory, but very recommended)
  - 10Gb/s single stream performance
  - use cases:
    - performance
    - fast speeds
    - low latency
- `Spread` – keep instances separated
  - limit of 7 instances per AZ
  - provides infrastructure isolation
  - each instance runs from a different rack
  - each rack has its own network and power source
  - not supported for dedicated instances or hosts
  - use case: small number of critical instances that need to be kept separated from each other (high availability)
- `Partition` - groups of instances spread apart
  - divided into partitions; max 7 per AZ
  - each partition has its own racks: no sharing between partitions
  - you can control which partition each instance will be placed into (or select auto-placement)
  - allow to contain the impact of failure to part of an application
  - use cases:
    - topology aware applications (see HDFS, HBase, Cassandra)
    - huge scale parallel processing systems

## EC2 Dedicated Hosts

EC2 Host dedicated to you. You pay for the host itself (no charges for the instances), which is designed for a specific family of instances (for example, a1, c5, m5, etc). On-Demand & Reserved options available. Host hardware has physical sockets and cores.

Examples (see [this article](https://aws.amazon.com/ec2/dedicated-hosts/pricing/) for more information):
- A1
  - 1 Socket
  - 16 Cores
  - can run only one instance type/size:
    - 16 medium
    - 8 large
    - 4 xlarge
    - 2 2xlarge
    - 1 4xlarge
- R5
  - 2 Socket
  - 48 Cores
  - can run different instance sizes:
    - 1 12xl + 1 4xl + 4 2xl
    - 4 4xl + 4 2xl

Limitations:
- AMI limits: RHEL, SUSE and Windows AMIs aren't supported
- Amazon RDS instances are not supported
- Placement groups are not supported

Hosts can be shared with other organization accounts using RAM (Resource Access Manager).

Often used to solve licensing issues.

## Enhanced Networking

Feature designed to improve the general performance of EC2 networking. Uses `SR-IOV` (Single Root I/O Virtualization); NIC is virtualization aware: Host Network Card provides each instance with different "logical network cards". Available on most EC2 types without any charge. Provides higher I/O, lower Host CPU usage, higher bandwidth, higher packets-per-seconds, consistent lower latency. More details can be found [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking.html).

## EBS Optimized

Per-instance binary option. Historically network was shared for data and EBS. EBS Optimized means dedicated capacity for EBS. Most instances support and have enabled by default. Some older instances may support, but for extra charge.

## Quiz

- What benefits does Enhanced Networking Provide (choose all that apply)
  - Higher Packets per Second (PPS)
  - Consistent Low Latency
  - High Throughput
- What placement group should be used when you need the best performance within EC2
  - Cluster
- What placement group is ideal when you need the best levels of resilience
  - Spread
- If you run a large application which uses 100's of EC2 instances and it needs exposure to physical location for performance and availability reasons. Which placement group should you use.
  - Partition
- How can permissions be provided to an application running in EC2 using best practices?
  - Instance Profile & IAM Role
- How many AZs can be used by a cluster placement group
  - 1
- How many instances can be within a spread placement group
  - 7 Per AZ
- What is true of dedicated hosts (pick 2)
  - There is no charge for EC2 instances running on a dedicated host
  - The host is dedicated to you
- Which feature of EC2 allows you to provide commands that the instance will run at startup
  - user-data
- When do commands specified in user-data get executed
  - Once when the instance is provisioned