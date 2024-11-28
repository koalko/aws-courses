# HA & Scaling

## Regional and global AWS architecture

### Overview

- Global Service Location and Discovery
- Content Delivery (CDN) and optimization
- Global health checks and failover
- Regional entry point
- Scaling and resilience
- Application services and components

### Global level

Globally DNS is used for:

- service discovery
- regional based health checks
- request routing

CDNs are used to cache content globally – as close to end users as possible to improve performance.

### Regional level

- Entry point: web tier (`ALB` / `APIGW`)
- Functionality: compute tier (`EC2` / `Lambda` / `ECS`)
- Raw data: storage tier (`EBS` / `EFS` / `S3`)
- Fast-access structured data: caching tier (`ElastiCache` / `DynamoDB Accelerator`)
- Persistent structured data: database tier (`RDS` / `Aurora` / `DynamoDB` / `Redshift`)
- Application services tier (`Kinesis` / `Step Functions` / `SQS` / `SNS`)

## Evolution of the Elastic Load Balancer

There are 3 load balancer (`ELB`) types available within AWS. You should avoid `v1` and prefer `v2`.

- `v1` (deprecated)
  - Classic Load Balancer (`CLB`): has been introduced in 2009; not really a layer-7 device (lacking features, 1 SSL certificate per CLB)
- `v2` (faster, cheaper, support target groups and rules)
  - Application Load Balancer (`ALB`): true layer-7 device; supports HTTP/HTTPS/WebSocket
  - Network Load Balancer (`NLB`): supports TCP, TLS, UDP

## Elastic Load Balancer Architecture

The job of a load balancer is to accept connections from customers and distribute those connections across any registered backend compute.

When you're creating an ELB, you specify network subnets to use (on subnet per AZ in 2+ AZs). When ELB is being provisioned, 1+ **Nodes** are placed into a subnet in each AZ. These nodes scale with load.

Each ELB is configured with an (A) record DNS name, which resolves to the ELB nodes. ELB can be internet-facing (nodes will have public IPs) or internal (nodes only have private IPs). Load balancer nodes are configured with listeners which accept traffic on a port and protocol, and communicate with targets on a port and protocol.

It is important to note that internet-facing LB nodes can access both public and private EC2 instances.

To properly function, LB needs `8+` free IPs (`/28`) per subnet; and it is recommended to use a `/27` or larger subnet to allow for scale.

It makes sense to have a LB for each scalable tier of an application (for example: web-tier, application-tier, etc.), so the tier-consumer will connect to the LB and not to the particular instance. This allows for scaling via abstracting underlying instances.

**Cross-Zone Load Balancing** allows to balance load evenly across AZs with different number of compute instances.

### Important tips on ELBs:

- ELB is a DNS A record pointing at 1+ Nodes per AZ
- Nodes (in one subnet per AZ) can scale
- Internet-facing LB: nodes have public IPv4 IPs
- Internal LB: nodes only have private IPs
- EC2 doesn't need to be public to work with a LB
- Listener configuration controls **what** the LB does
- LB requires 8+ free IPs per subnet, and `/27` subnet to allow scaling

## ALB vs NLB

Classic Load Balancer (CLB, `v1`) requires separate setup per domain name (only one SSL-certificate per CLB is supported).

`v2` load balancers support rules and target groups. Host based rules using SNI (Server Name Indication) and an ALB allows consolidation.

Application Load Balancer (ALB):

- is a **Layer 7** LB, which listens on HTTP and/or HTTPS
- doesn't support other Layer 7 protocols (SMTP/SSH/...)
- doesn't support TCP/UDP/TLS listeners
- can understand: L7 content type, cookies, custom headers, user location and app behaviour
- HTTPS (SSL/TLS) always terminated on the ALB (**no unbroken SSL**, a new connection is made to the application)
- ALBs **must** have SSL certificates if HTTPS is used
- ALBs are slower than NLBs: more levels of the network stack to process
- health checks evaluate application health (layer 7)

ALB Rules affect direct connections which arrive at a listener. Rules are processed in priority order; last one to process is a default rule (catchall). Rule conditions: host-header, http-header, http-request-method, path-pattern, query-string, source-ip. Rule actions: forward, redirect, fixed-response, authenticate-oidc, authenticate-cognito.

Network Load Balancer (NLB):

- is a Layer 4 LB (TCP/TLS/UDP/TCP_UDP)
- no visibility or understanding of HTTP or HTTPS
- no headers, no cookies, no session stickness
- fast (millions of rps, 25% of ALB latency)
- good for non-HTTP(S) protocols (SMTP/SSH/etc.)
- health checks just check ICMP/TCP handshake (not app-aware)
- can have static IPs (useful for whitelisting)
- can forward TCP to instances (unbroken encryption)
- used with PrivateLink to provide services to other VPCs

Reasons to use NLB are:

- unbroken encryption
- static IP for whitelisting
- fastest performance
- non-HTTP(S) protocols
- PrivateLink

## Launch Configurations and Launch Templates

### Key concepts

Both Launch Configurations (`LC`) and Launch Templates (`LT`) allow to define the configuration of an EC2 instance in advance: AMI, Instance Type, Storage, Key Pair, Networking, Security Groups, Userdata, IAM Role.

Both are not editable (defined once). LT has versions. LT provide newer features: T2/T3 Unlimited, Placement Groups, Capacity Reservations, Elastic Graphics.

Both LC and LT can be used with Auto Scaling Groups, but LT can also be used to provision separate EC2 Instances.

## Auto Scaling Groups (ASGs)

### General

ASGs:

- provide automatic scaling and self-healing for EC2
- uses Launch Templates or Launch Configurations
- has **minimum**, **desired** and **maximum** size (example: `1:2:4`)
- keep running instances at the desired capacity by provisioning or terminating instances
- has Scaling Policies to automatically adjust the **desired** capacity between **min** and **max** values based on metrics

You specify subnets to use with ASG. AWS will attempt to keep the number of instances within each AZ (within ASG) even.

ASGs monitor the health of an instance they provision (by default using EC2 Status Check).

ASG instances are automatically added to or removed from the target group. ASG can use the Load Balancer health checks (rather than EC2 status checks), which provides application awareness.

### Scaling Policies

ASGs don't **need** scaling policies; they can have none (manual scaling).

- Manual Scaling: manually adjust the desired capacity
- Scheduled Scaling: time-based adjustment
- Dynamic Scaling
  - Simple (examples: "CPU above 50% +1", "CPU below 50% -1")
  - Stepped (bigger +/- based on a difference; great for variable load; better to use it than simple scaling)
  - Target Tracking (example: desired aggregate CPU = 40%)
  - Scaling Based on `SQS`: `ApproximateNumberOfMessagesVisible`
- Cooldown Periods: allows to wait after previous change

### Scaling Processes

- `Launch` and `Terminate`: Suspend and Resume
- `AddToLoadBalancer`: add to LB on launch
- `AlarmNotification`: accept notification from CloudWatch
- `AZRebalance`: balances instances evenly across all of the AZs
- `HealthCheck`: instance health checks on/off
- `ReplaceUnhealthy`: terminate unhealthy and replace
- `ScheduledActions`: scheduled on/off
- `Standby`: to suspend any activites of the ASG on a specific instance

### Useful tips

- ASGs are free (only the resources created are billed)
- use cooldowns to avoid rapid scaling
- use smaller instances in greater numbers (for granularity)
- use with ALB for elasticity (for abstraction)
- ASG defines **when** and **where**; Launch Template defines **what**

## ASG Lifecycle Hooks

ASG Lifecycle Hooks are Custom Actions on instances during ASG actions (**instance launch** or **instance terminate** transitions). When Lifecycle Hooks are created, instances are paused within the flow. They wait either for a configurable timeout (then either `continue` or `abandon` ASG action), or for `CompleteLifecycleAction` triggered to resume the ASG process. Lifecycle Hooks can be integrated with EventBridge or SNS Notifications, which allow your system to perform event-driven processing based on the launch or termination of EC2 instances within the ASG.

## ASG Health Checks

ASG assess the health of instances within the group using health checks; if the instances fails a health check, it is replaced withing the group. Three types of health checks, which can be used with ASG:

- EC2 (default)
  - Unhealthy statuses: `stopping`, `stopped`, `terminated`, `shutting down`, `impaired` (not 2/2 status)
- ELB (can be enabled)
  - Healthy if the status is `running` and instance is passing ELB health check
  - Can be more application aware
- Custom
  - Instances marked `healthy` and `unhealthy` by an external system

Health check grace period (default **300s**) – delay before starting checks. Allows to account for a system launch, bootstrapping and application start.

## SSL Offload

There are three ways that a load balancer can handle secure connection:

- Bridging (default) (ELB)
  - One or more clients make one or more connections to load balancer
  - Listener is configured for HTTPS
  - Connection is terminated on the LB & needs a certificate for the domain name
  - LB initiates a new SSL connection to backend instances
  - Instances need SSL certificates and the compute required for cryptographic operations
  - Pros:
    - LB can see the request contents and make decisions based on those
  - Cons:
    - Certificates needs to be stored both on LB and Instances (admin overhead)
    - AWS has access to certificate (might be a security policy issue)
    - Instances need to perform cryptographic operations (extra compute)
- Pass-through (NLB)
  - Listener is configured for TCP
  - LB just passes the client connection to one of the backend instances
  - Connection encryption is maintaned between the client and backend instances
  - No encryption or decryption happens on the LB
  - Each instance needs to have the appropriate SSL certificate installed
  - Pros:
    - AWS (LB) doesn't have access to certificate (and encrypted HTTP data)
  - Cons:
    - Impossible to load balance based on HTTP data
    - Each instance need to have certificate installed (admin overhead)
    - Instances need to perform cryptographic operations (extra compute)
- Offload (ELB)
  - Listener is configured for HTTPS
  - Connection is terminated on LB
  - Backend connections use HTTP
  - Pros:
    - Data is still encrypted between the client and AWS (LB)
    - No need to manage certificates on instances
    - No need to perform cryptographic operations on the instances
  - Cons:
    - AWS has access to certificate (might be a security policy issue)
    - Data is in plaintext inside the AWS network (might be a security policy issue)

## Connection Stickiness

Required then user session state is stored on the instances (rather than in some shared database). Session Stickiness can be enabled on a LB, so:

- a first time user makes a request, the LB generates a cookie (`AWSALB`), which locks the client to a single backend instance for a duration (ranging from 1second to 7days; can be defined when enabling a feature)
- if a request already have this cookie, LB will redirect the request to the backend instance client has been locked in earlier

Potential problem: this can cause an uneven load of a backend server. So, it is preferrable to use an external shared session data storage instead (if possible).

## Gateway Load Balancer (GWLB)

Helps to run and scale 3rd-party appliances: firewalls, intrusion detection and prevention systems, data analysis tools, etc. Allows for transparent inspection and protection of inbound and outbound traffic.

Has two major components:
- GWLB endpoints (`GWLBE`): traffic enters/leaves via these endpoints
- `GWLB` itself: balances across multiple backend appliances

Traffic and metadata is tunneled using `GENEVE` protocol.

Simplified algorithm for incoming packets:
- client accessing the web site
- ingress (gateway) route table directs any traffic destined for the ALB subnets at the `GWLBE` in the same AZ
- `GWLBE` routes traffic to GWLB, which is usually resides with a separate security VPC
- packets are encapsulated (to allow routing in different VPC) and sent through to the appliances
- afterwards, packets recieved from appliances transferred back to `GWLBE`, which sents them to ALB

Outgoing packets follow the same route backwards.

## Quiz

- Which load balancer is capable of handling 1,000,000's of requests per second
  - Network Load Balancer
- Which load balancer is capable of handling traffic other than HTTP and HTTPS
  - Network Load Balancer
- What is SSL Offload
  - HTTPS to the load balancer, HTTP to the instances
- What is true of Launch Configuration and Launch Templates (choose all that apply)
  - Launch templates can be used to directly launch instances
  - Launch Templates support versioning
- What 3 values determine how auto-scaling-groups operate
  - Min Size
  - Max Size
  - Desired Capacity
- Where is the AMI defined ...
  - LC/LT
- Where is the user-data defined
  - LC/LT
- Where are scaling policies defined
  - Optionally in ASG
- Which load balancer is allocated with a static IP
  - NLB
- Which load balancer can take decisions based on HTTP paths
  - ALB