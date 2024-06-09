# IAM, Accounts and AWS Organizations

## Identity policies

IAM Identity policy is a set of security statements. IAM policy document is one or more statements. Each statement grants or denies access to AWS resources for the identity the policy attached to.

- `Sid`: statement ID (optional field describing the statement)
- `Effect`: `Allow` or `Deny`
- `Action`: one or more actions (format is `<service>:<operation>`; `<service>:*` for a wildcard action)
- `Resource`: one or more resources (`arn`s, optionally with wildcard)

You can allow a wildcard action on a wildcard resource and then deny some specific actions on a specific resources. Explicit `Deny` wins.

The priority:

1. Explicit `Deny`
2. Explicit `Allow`
3. Implicit `Deny`

Ways to apply policies to identities:

- inline: fill and apply policy to each identity directly
- managed: create policy once and apply the same policy to a set of identities

IAM Users are an identity used for anything requiring long-term AWS access (humans, applications, service accounts).

`Principal`: individual people, services, computers, or a group of any of previously mentioned entities. Principal makes a request to IAM to interact with resources. The first step is an `authentication` (to prove that a principal is a registered identity). Authentication could be performed via username/password or via access keys. After successful authentication `principal` becomes an `authenticated identity`. When an authenticated identity tries to do something with AWS resources, IAM checks that the identity is `authorized` to perform corresponding actions via the `authorization` process.

Amazon Resource Name (ARN): uniquely identify resources within any AWS accounts. ARN might contain wildcards. Format examples:

```
arn:partition:service:region:account-id:resource-id
arn:partition:service:region:account-id:resource-type/resource-id
arn:partition:service:region:account-id:resource-type:resource-id
```

ARN examples:

- `arn:aws:s3:::catgifs`: a specific bucket (not the contents)
- `arn:aws:s3:::catgifs/*`: everything in a specific bucket, but not the bucket itself

Not specifying region and specifying `*` are a different things.

Partition is usually `aws` (other examples: `aws-cn` for China).

Service is AWS product, for example: `s3`, `iam`, `rds`.

Region is an AWS region. Often omitted.

Account ID is an AWS Account ID.

Resource ID depends on a resource type.

Some IAM limits:

- 5000 IAM users per account
- IAM user can be a member of 10 groups

## IAM Groups

IAM Groups are just containers for Users. You cannot login into the IAM Group. User can be a member of any (up to 10) number of groups. Groups can have policies (inline/managed) attached to them. There is no default "All Users" group in AWS. You can't nest groups within groups. There is a soft limit of 300 groups per AWS account (can be increased through support ticket). AWS Group is **not a true identity**. It cannot be referenced as a `principal` in a policy.

## IAM Roles

IAM Role used by an (unknown) number of principals (users/applications/services). IAM Roles are assumed (for a time): a principal become that role. You can say that a principal borrows the permissions for a time using the IAM Role. AWS generates temporary security credentials when the role is being assumed. These temporary security credentials are generated by the AWS STS: `Secure Token Service` (operation is `sts:AssumeRole`).

IAM Role have two types of policies which could be attached to it:

- trust policy: controls which identities could assume the role
- permissions policy: controls which resources could be accessed by assuming the role

Roles are **real identities**. They can be referenced as a `principal` in a policy.

### Examples of IAM Roles usage

#### `AWS Lambda`

There is a `Lambda Execution Role`, which trusts the AWS Lambda service, and have a permissions policy to allow it to interact with AWS resources. Runtime environment assumes the role using STS, gets the temporary security credentials, and interacts with AWS resources.

#### Rarely needed (emergency) access

Special Emergency Role to use only in exceptional cases.

#### External identity provider

When you need to arrange AWS Account access for users from an external identity provider (for example, Active Directory), you can use IAM Roles so that external accounts are able to assume the role with required permissions. Enabling external provider to assume the IAM Role called the `ID federation`.

#### Application with millions of users

If an application with millions of users needs to use AWS resources (for example, `DynamoDB`), you can give permission for the web identities of this application to assume the role in AWS account. This process called `web identity federation`. Pros:

- no AWS credentials stored in the application
- uses existing accounts
- can be scaled to hundreds of millions of user accounts

#### Cross-account access

Allow users from one AWS account to assume the role inside other AWS account to provide access to necessary resources.

## Service-linked roles

Service-linked role is an IAM role linked to a specific AWS service (and pre-defined by this service), providing permissions that a service needs to interact with other AWS services on your behalf. Service might create/delete the role or allow you to do so during the setup or within IAM. You can't delete the service-linked role until it's no longer required.

Example of a policy which allows to create aa service-linked roles:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iam:CreateServiceLinkedRole",
      "Resource": "arn:aws:iam::*:role/aws-service-role/SERVICE-NAME.amazonaws.com/SERVICE-LINKED-ROLE-NAME-PREFIX*",
      "Condition": {
        "StringLike": { "iam: AWSServiceName": "SERVICE-NAME.amazonaws.com" }
      }
    },
    {
      "Effect": "Allow",
      "Action": ["iam:AttachRolePolicy", "iam: PutRolePolicy"],
      "Resource": "arn:aws:iam::*:role/aws-service-role/SERVICE-NAME.amazonaws.com/SERVICE-LINKED-ROLE-NAME-PREFIX*"
    }
  ]
}
```

See [this article](https://docs.aws.amazon.com/IAM/latest/UserGuide/using-service-linked-roles.html) for details.

### Passing a role

Example of a policy statement with allows a principal to pass a role to an AWS Service:

```json
{
  "Effect": "Allow",
  "Action": ["iam:ListRoles", "iam:PassRole"],
  "Resource": "arn:aws:iam::123456789012:role/my-role-for-XYZ"
}
```

## AWS Organizations

You use a standard AWS account to create an AWS Organization. AWS Organization isn't created within the account. Account, which was used to create an organization, called `Management Account` (previously `Master Account`). You can invite other existing standard AWS accounts to join the organization. After an AWS account joined the organization, it becomes a `Member Account`. Organization can have only one `Management Account` and any number of `Member Account`s.

AWS Organization is a tree with an `Organization Root` at the root of the tree. `Organization Root` can contain AWS accounts or `Organizational Unit`s (`OU`s), which can contain either AWS accounts or other `OU`s.

AWS Organization has a `Consolidated Billing` feature: member accounts can pass their billing to the management account ("payment account" in that case).

AWS Organization provides consolidation of reservations and volume discounts, which allows to reduce infrastructure costs.

AWS Organization features service called `Service Control Policies` (`SCP`), which allows to restrict what AWS accounts within the AWS organization can do.

You can create new AWS Accounts directly within the AWS Organization.

You can use IAM users from the single account (management or member) to access resources in other accounts within the organization using IAM roles. Alternatively, you can use identity federation to use external identities to access AWS resources within the organization. This allows to avoid overhead of managing IAM users in each AWS account.

## Service Control Policies

Feature to restrict AWS Accounts. SCP is a JSON document, which can be attached to:

- `Organization Root`
- `Organizational Unit`
- AWS Account

SCP inherits down the organization tree (but not when attached to an AWS Account). Even when `Management Account` have SCP attached, the account itself is not affected by these policies (regardless of how SCP attached). It might make sense to not use management account for creating AWS resources because of this limitation.

SCP are account permission boundaries. They limit what the account (including account root user) can do. SCP don't grant any permissions.

Two ways of using SCP:

- allow list: block by default and allow certain services
- deny list: allow by default and deny certain services

If SCP is enabled, there is implicit global `Deny`.

FullAWSAccess policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

DenyS3 policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}
```

AllowS3EC2 policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*", "ec2:*"],
      "Resource": "*"
    }
  ]
}
```

Access permissions determined by intersection of SCP and Identity Policies in Accounts.

## CloudWatch Logs

Public Service, usable from AWS or on-premises.

Store, monitor and access logging data.

AWS Integrations: EC2, VPC Flow Logs, Lambda, CloudTrail, R53, ...

Can generate metrics based on logs (metric filter).

CloudWatch is a regional service.

Logging sources inject logs into CloudWatch as Log Events (timestamp+message).

`Log Stream`: log events from the same source.

`Log Group`: container of log streams of one type. Log group also stores configuration settings (retention/permissions/etc.). Metric filters are also defined on log groups.

## CloudTrail

CloudTrail is a product which logs API calls and account events.

Logs API calls/activities as a `CloudTrail Event`.

It's very often used to diagnose security or performance issues, or to provide quality account level traceability.

It is enabled by default in AWS accounts and logs free information with a 90 day retention. This information is being stored in `Event History`.

It can be configured to store data indefinitely in S3 or CloudWatch Logs. To customize the service create one or more `Trails`.

CloudTrail Event types:
- `Management Events`: information about management operations performed on resources in AWS Account (control plane operations); example: EC2 instance created
- `Data Events`: information about resource operations performed on resources; example: object uploaded to S3
- `Insight Events`: unusual activity, errors, or user behavior in your account

By default only `Management Events` are being logged.

You can create a new `Trail`:
- per one region
- for all regions

Most services log events in the region where event occurs. Very small number of services log events globally (into `us-east-1` actually; examples of such services are `IAM`, `STS`, `CloudFront`): global service events.

When you create a `Trail`, both `Management Events` and `Data Events` can be captured. By default only `Management Events` are enabled; `Data Events` should be enabled specifically (and will incur extra costs).

CloudTrail can be integrated with CloudWatch Logs.

You can create Organizational Trail, which will store all events from all accounts in the organization.

CloudTrail is **not** realtime (logs might be delivered in ~15 minutes after even occurs).

One `Trail` for `Managed Events` per region is free.

`CloudTrail` -> `Event history` is being populated even if no Trails had been created.

## AWS Control Tower

Quick and easy setup of multi-account environment.

Orchestrates other AWS services to provide this functionality.

Uses Organizations, IAM Identity Center, CloudFormation, Config, ...

SSO/ID Federation, centralized logging and auditing.

Basically an evolution of Organizations.

Base part of Control Tower is called `Landing Zone`: multi-account environment.

Other parts:
- `Guard Rails`: detect/mandate rules/standards across all accounts.
- `Account Factory`: automates and standardizes new account creation.
- `Dashboard`: single page oversight of the entire environment.

When `Management Account` setup via `Control Tower`, the following OUs created:
- `Foundational OU` (security)
  - `Audit` Account
    - `SNS`
    - `CloudWatch`
  - `Log Archive` Account
    - `AWS Config`
    - `CloudTrail`
- `Custom OU` (sandbox)
  - `Account Factory` will create additional accounts here using:
    - Account Baseline template
    - Network Baseline template

`Control Tower` uses `CloudFormation` for automation.

`Control Tower` uses `Config` and `SCP`s to implement account `Guard Rails`.

### `Control Tower`: `Landing Zone`

Well-architected multi-account environment; have a home region. Landing zone is built with `Organizations`, `Config`, `CloudFormation`. Security OU: log archive, audit accounts. Sandbox OU: test/less rigid security. You can create other OUs and Accounts. Utilizes IAM Identity Center (AWS SSO), multiple accounts, ID Federation. Monitoring and notifications using `CloudWatch` and `SNS`. End User account provisioning using `Service Catalog`.

### `Guard Rails`

Rules for multi-account governance. Mandatory, strongly recommended or elective. Functions in three ways:
- `preventive`: stops from doing things (AWS Organization SCP)
  - enforced or not enabled
  - example: allow or deny regions or disallow bucket policy changes
- `detective`: compliance checks (AWS Config Rules)
  - `clear`, `in violation` or `not enabled`
  - example: detect `CloudTrail` enabled or EC2 Public IPv4

### `Account Factory`

Automated account provisioning using cloud admins or end users (with appropriate permissions). `Guard Rails` automatically added. Account admin given to a named user (`IAM Identity Center`). Account & network standard configuration. Accounts can be closed or repurposed. Can be fully integrated with a business SDLC (software development lifecycle) via APIs.