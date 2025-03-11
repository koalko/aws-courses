# Security, deployment and operations

## AWS Secrets Manager

Shares functionality with Parameter Store, but designed specifically for secrets (passwords, API keys, etc.). Usable via Console, CLI, API or SDKs. Designed to be integrated inside other applications. Supports automatic rotation (via periodically invoked lambda function). Directly integrates with some AWS products (for example, RDS).

An example of Secrets Manager usage:

- Secrets Manager controls the RDS database credentials
- lambda function periodically rotates (via IAM roles) the database credentials both in Secrets Manager and in RDS
- application uses SDK (via IAM credentials authorization) to access the database credentials and connect to the database

Secrets are encrypted using KMS.

## Application Layer (L7) Firewall

Layer 3/4 firewall (understands layers 1-4) sees only packets and segments; request and response are different and unrelated.

Layer 5 firewall (understands layers 1-5) has the concept of a session, so request and response can be considered as part of one session. This reduces admin overhead and allows for more contextual security.

Layer 7 firewall (understands layers 1-7) is aware of the Layer 7 protocol (HTTP). It can identify normal or abnormal requests (and, for example, protect against protocol-specific attacks).

Usually, HTTPS connection is terminated (decrypted) on the L7 firewall, and a new encrypted HTTPS connection is established between the firewall and the backend. So, L7 firewall has an ability to analyze HTTPS traffic. Data at L7 can be inspected and blocked, replaced or tagged (adult/spam/offtopic/etc.).

L7 firewall keeps all L3, L4 and L5 features, but can also react to L7 elements, such as DNS information, requests rate, content, headers, etc.

## Web Application Firewall (WAF), WEBACLs, Rule Groups and Rules

Some services, which support WAF:

- CloudFront
- ALB
- AppSync
- API Gateway

The unit of configuration within the WAF is `Web ACL`. Within a Web ACL you have `Rule Groups`, and Rule Groups consist of `Rules`. Web ACL can be updated manually; but you can also use automation, for example: `EventBridge` and scheduled rules to parse public IP lists to block known bad actors.

WAF output logs, which can be directed to S3 (delivered every 5 minutes), CloudWatch Logs and Kinesis Firehose. You can use automated logs processing (via Lambda, for example) to tweak some Web ACL rules without human interaction.

Web ACL default action (allow or block) will be applied for non-matching traffic.

When you create Web ACL for a regional service (ALB, APIGW, AppSync), you need to pick a region for a Web ACL.

Rule groups and rules are processed in order. Web ACL Capacity Units (`WCU`) used to indicate the complexity of rules: default is 1500 (can be increased with a support ticket). Web ACLs are associated with resources (this can take time); **adjusting** a Web ACL takes less time than **associating** one. One resource can have one Web ACL, but one Web ACL can be associated with many resources (but you can't associate CloudFront Web ACL with regional resource and vice versa).

Rule groups contain rules. They don't have default actions. Groups can be:

- managed (by AWS or marketplace)
- yours
- service owned (Shield or Firewall Manager)

The same rule group can be referenced by multiple Web ACLs. Rule group also have a WCU capacity (defined upfront, max 1500).

WAF rule structure:

- type
  - regular (something occurs); example: allow SSH traffic from specific IP
  - rate-based (something occurs at a certain rate); example: 5000 attempted SSH connections from IP per 5 minutes
- statement: single or multiple, connected with `and`, `or`, `not`
  - what to match; example: incoming SSH or specific header
    - origin country
    - IP
    - label
    - header
    - cookies
    - query parameter
    - URI path
    - query string
    - body (first 8192 bytes)
    - HTTP method
  - count all
  - what and count
- action
  - allow (cannot be applied for rate-based rules)
  - block
  - count
  - captcha
  - custom response (`x-amzn-waf-...`)
  - label (can be referenced later in the same Web ACL to perform multi-stage flow)

With `allow` and `block` actions no further WAF action can occur. With `count` and `captcha` actions processing continues.

Pricing:

- monthly ($5/month) per Web ACL
- monthly ($1/month) per rule on Web ACL
- monthly for rule group
- monthly ($0.60/month) per 1 million requests
- optional security features:
  - intelligent threat mitigation
  - bot control: $10/month and $1 per 1 million requests
  - captcha: $0.40 per 1000 challenge attempts
  - fraud control / account takeover: $10/month and $1 per 1000 login attempts
- marketplace rule groups

## AWS Shield

Provide DDOS protection. Protects against three types of attacks:

- network volumetric attacks (L3) (saturate capacity)
- network protocol attacks (L4) (TCP SYN flood)
- application layer attacks (L7) (web request flood)

Comes in two forms: standard and advanced.

### AWS Shield Standard

- free for AWS customers
- protection at the perimeter (region/VPC or AWS edge)
- protects against common network (L3) or transport (L4) layer attacks
- best protection using:
  - R53
  - CloudFront
  - Global Accelerator

### AWS Shield Advanced

- has a cost: $3000 per month (per org), 1 year lock-in + data out per month
- protects:
  - R53
  - CloudFront
  - Global Accelerator
  - anything associated with EIPs (for example, EC2)
  - ALBs
  - CLBs
  - NLBs
- not automatic: must be **explicitly** enabled in Shield Advanced or AWS Firewall Manager' Shield Advanced policy
- cost protection (for example, EC2 scaling) for unmitigated attacks
- proactive engagement and access to AWS Shield Response Team (SRT)
- features:
  - WAF integration: includes basic AWS WAF fees for web ACLs, rules and web requests
  - Application Layer (L7) DDOS protection (uses WAF)
  - real-time visibility of DDOS events and attacks
  - health-based detection: application-specific health checks, used by proactive engagement team
  - protection groups (for resources)

## CloudHSM

HSM – Hardware Security Module.

Similar to KMS: an appliance which creates, manages and secures cryptographic material (keys). KMS is a shared service, which uses HSM under the hood. CloudHSM is a true "single tenant" HSM, which is provisioned by AWS, but fully managed by a customer.

Fully `FIPS 140-2 Level 3` (KMS is `Level 2` overall, _some_ capabilities are `Level 3`). Less integrated with AWS services than KMS.

Supports industry standard APIs:

- `PKCS#11`
- Java Cryptography Extensions (`JCE`)
- Microsoft `CryptoNG` (CNG) libraries

KMS can use `CloudHSM` as a **custom key store**, which provides indirect CloudHSM integration with AWS.

HSM by default is not a HA device, so you need to create an HSM cluster (with HSM devices in different AZs) to achieve HA. HSMs keep keys and policies in sync when nodes are added or removed. HSMs operate in an **AWS managed HSM VPC**. Interfaces are added to **customer VPC** (one `ENI` per `HSM`). AWS CloudHSM client needs to be installed on EC2 instances in order to work with HSMs.

AWS provision HSM, but have no access to secure area where key material is held.

### Key points on CloudHSM

- no native AWS integration (e.g. no S3 SSE)
- can be used to:
  - offload the SSL/TLS processing for web servers
  - enable transparent data encryption (TDE) for Oracle databases
  - protect the private keys for an issuing Certificate Authority (CA)

## AWS Config

Main job of AWS Config is to record configuration changes over time on resources. AWS Config is great for auditing of changes and checking that settings are compliant with standards; but it does not prevent changes happening. Regional service; supports cross-region and account aggregation. Changes can generate SNS notifications and near-realtime events via EventBridge & Lambda.

Once enabled, the configuration of all supported resources is constantly tracked. Every time a change occurs to a resource, a `configuration item` (`CI`) is created for that resource. A `CI` represents the **configuration of a resource at a point in time** & its **relationships**. All CI's for a given resource is called a `configuration history` and is stored in S3.

Resources are evaluated against `Config Rules`: either AWS managed, or custom (using lambda). EventBridge can invoke lambda functions based on AWS Config events for automatic resource remediation.

## Amazon Macie

Data security and data privacy service. Can be used to discover, monitor and protect data, stored in S3 buckets. Supports automated discovery of data, i.e. `PII` (personally identifiable information), `PHI` (personal health information), `Finance`, etc. using data identifiers (rules, which content is assessed against).

Two types of data identifiers:

- managed
  - built-in
  - ML and pattern matching
- custom
  - proprietary
  - regex-based
  - supports `keywords`:
    - optional sequences that need to be in proximity to regex match
    - supports `Maximum Match Distance` – how close keywords are to regex pattern
  - supports `ignore words`:
    - if regex match contains ignore words, it's ignored

With Macie you create discovery jobs, which use data identifiers and look for anything matching on buckets. If anything is found, these jobs generate findings, which can be viewed interactively, or can be used as a part of integration with other AWS services (e.g. `Security Hub` or `EventBridge`).

Macie uses multi-account architecture; it is centrally managed either via AWS ORG or one Macie account inviting other accounts.

Macie supports two types of findings:

- policy findings
  - generated when the bucket access policies has been "relaxed"
  - can only be generated after Macie has been enabled
  - examples:
    - Policy:IAMUser/S3BlockPublicAccessDisabled
    - Policy:IAMUser/S3BucketEncryptionDisabled
    - Policy:IAMUser/S3BucketPublic
    - Policy:IAMUser/S3BucketSharedExternally
- sensitive data findings
  - examples:
    - SensitiveData:S3Object/Credentials
    - SensitiveData:S3Object/CustomIdentifier
    - SensitiveData:S3Object/Financial
    - SensitiveData:S3Object/Multiple
    - SensitiveData:S3Object/Personal

## Amazon Inspector

Scans EC2 instances, the instance OS and containers for vulnerabilities and deviations against best practice. An assessments running time can differ: 15 minutes, 1 hour, 8/12 hours, 1 day. Provides a report of findings (security report) ordered by priority (severity).

Inspector can work with two different types of assessments:

- network assessment (agentless)
- network and host assessment (agent)

Rules packages determine what is checked.

For network reachability no agent is required (but agent can provide additional OS visibility). Checks reachability end to end:

- EC2
- ALB
- DX
- ELB
- ENI
- IGW
- ACLs
- RTs
- SGs
- Subnets
- VPCs
- VGWs
- VPC Peering

Findings:

- RecognizedPortWithListener
- RecognizedPortNoListener
- RecognizedPortNoAgent
- UnrecognizedPortWithListener

For host assessments the agent is required:

- common vulnerabilities and exposures (`CVE`)
- center for internet security (`CIS`) benchmarks
- security best practices for Amazon Inspector

## Amazon Guardduty

**Continuous** security monitoring service. Analyzes supported data sources. Uses AI/ML and threat intelligence feeds. Identifies unexpected and unauthorized activity. Able to notify (usually via SNS) or trigger an event-driven protection/remediation (usually via lambda). Supports multiple accounts (`master` and `member` account architecture).

Supported data sources:

- Route53 DNS Logs
- VPC Flow Logs
- CloudTrail Event Logs
- CloudTrail Management Events
- CloudTrail S3 Data Events

## Quiz

- Shield Standard is automatically provided with the following services
  - CloudFront
  - R53
- Shield protects against what type of attack
  - DDOS
- WAF Provides what type of protections
  - Layer 7 attacks
  - SQL Injection
  - Cross-Site Scripting
- WAF Can be added to ...
  - CloudFront
  - APIGateway
  - ALB
- The main feature which Secrets Manager provides over SSM Parameter store is..
  - Password Rotation