# Network storage & data lifecycle

## Elastic File System (EFS) Architecture

EFS is an implementation of NFSv4 (Network File System v4). EFS filesystem can be mounted in Linux (only) and can be shared between many EC2 instances. Private service, accessible via mount targets inside a VPC (but can be accessed from on-premises via VPN or Direct Connect). Mount targets have IPs from VPC subnet range. To ensure HA it is advised to have mount targets in different AZs.

Performance modes:

- general purpose (default, suitable for latency-sensitive applications)
- max I/O (higher levels of throughput and op/s, suitable for highly parallel applications)

Throughput modes:

- bursting (like GP2 in EBS, throughput scales with a size )
- provisioned (like IO1 in EBS, you can specify throughput independently from size)

Storage classes:

- standard
- infrequent access (IA, lower cost)

Lifecycle policies can be used to switch between storage classes.

Example commands to mount EFS storage on Linux:

```sh
sudo mkdir -p /efs/wp-content
sudo dnf -y install amazon-efs-utils
sudo nano /etc/fstab
# Add the line:
#   file-system-id:/ /efs/wp-content efs _netdev,tls,iam 0 0
sudo mount /efs/wp-content
```

## AWS Backup

Fully managed data protection (backup/restore) service. Allows to consolidate management into one place across accounts and across regions. Supports a wide range of AWS products:

- compute (EC2/VMWARE)
- block storage (EBS)
- file storage (EFS, FSx)
- databases (Aurora, DynamoDB, Neptune, RDS, DocumentDB)
- object storage (S3)

**Backup plans** allow to configure:

- frequency
- window
- lifecycle
- vault
- region copy

**Backup resources** determines what resources are backed up.

**Vaults** are backup destinations (containers). Can be encrypted via KMS.

You can enable vault lock: write-once, read-many (WORM), 72 hour cool off, then even AWS can't delete (designed for compliance support).

You can create backups on-demand (manual backups).

Some services (S3, RDS) support point-in-time recovery (PITR).

## Quiz
- What protocol does EFS use
  - NFS
- What Operating systems does EFS Support
  - Linux only
- Is EFS a public or private AWS Service
  - Private
- Where is EFS Accessible from
  - Inside a VPC or any on-premises locations connected to that VPC
- What type of use-case is EFS ideally suited for
  - Shared storage across linux instances
