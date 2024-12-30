# Serverless and application services

## Architecture

Monolithic architecture: application implemented and deployed as a single "blob". Everything fails together, scales together and bills together.

Tiered architecture: different (in responsibilities) application parts separated into tiers. Every tier can be scaled independently. But tiers are still highly coupled: synchronous communication; no ability to scale to zero.

Event-based architecture: similar to tiered architecture, but with cross-tier communication made asynchronous using queues. Auto Scaling Group can be set to scale (desired count ) depending on the queue length. Tiers are completely decoupled from each other and can scale independently from zero.

Microservice architecture: similar to tiered architecture, but the responsibility area of each microservice is as small as possible (while still making sense). Microservice (in regards to events/messages) can be consumer, producer or both.

AWS has an `Event Router` service. It allows different services to use a single event bus, while routing events based on pre-defined rules.

Some notes on an event-based architecture:

- no constant running
- producers generate events when something happens
- events are delivered to consumers
- actions are taken and the system returns to waiting
- mature (?) event-driven architecture only consumes resources while handling events

## AWS Lambda

Function-as-a-Service (`FaaS`): short-running and focused. Lambda function – a piece of code lambda runs. Functions use a runtime (e.g. Python 3.8). Functions are loaded and run in a runtime environment. The environment has a direct memory (and an indirect CPU) allocation. You are billed for the duration that a function runs. Lambda is a key part of serverless architectures.

Common lambda runtimes:

- Python
- Ruby
- Java
- Go
- C#

You can request from 128MB to 10240MB (with 1MB steps) of RAM for lambda. 1769MB of memory gives you 1vCPU. 512MB storage available as `/tmp` (up to 10240MB).

Lambda function can run for up to **900 seconds** (**15 minutes**, function timeout).

Security for lambda functions is controlled via execution roles.

Common uses for lambda functions:

- serverless applications (`S3`, `API Gateway`, `Lambda`)
- file processing (`S3`, `S3 Events`, `Lambda`)
- database triggers (`DynamoDB`, `Streams`, `Lambda`)
- serverless cron (`EventBridge` / `CloudWatch Events`, `Lambda`)
- realtime stream data processing (`Kinesis`, `Lambda`)

By default lambda functions are given public networking. They can access public AWS services and the public internet. Such lambda functions have no access to VPC-based services unless public IPs are provided and security controls allow external access.

Lambda function can be configured to run in a private subnet inside a VPC. Such lambda functions can access VPC resources (with proper networking rules in order), but extra steps needs to be made for a function to access external resources. For example, to access DynamoDB you can set up a VPC endpoint; to access an external API you can set up a `NATGW` and an `IGW`.

Earlier private lambdas each used a separate ENI in a subnet. Now all lambdas using the same subnet and the same security group will use the single ENI. ENI creation/setup requires around 90s, but it is done once (per subnet/security group).

Lambda function can use `execution role` (similar to EC2 instance role): it is an IAM role, attached to lambda function, which control the **permissions** the lambda function **receives**.

An example of execution role policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": ["ec2:Start*", "ec2:Stop*"],
      "Resource": "*"
    }
  ]
}
```

Example of a trust policy for lambda function:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": ["sts:AssumeRole"],
      "Effect": "Allow",
      "Principal": {
        "Service": ["lambda.amazonaws.com"]
      }
    }
  ]
}
```

Lambda resource policy controls **what** services and accounts can **invoke** lambda function.

Lambda uses `CloudWatch` (for metrics like invocation success/failure, retries, latency), `CloudWatch Logs` (for logs from executions) and `X-Ray` (for distributed tracing). Using `CloudWatch Logs` requires permissions via `execution role`.

Types of lambda invocations:

- **Synchronous invocation**. Client calls the function (optionally through APIGW) and waits for the result/error.
- **Asynchronous invocation**. Typically used when AWS services invoke lambda functions. If processing of the event fails, lambda will retry between 0 and 2 times (configurable); hence the lambda function needs to be idempotent. Events can be sent to dead letter queues (DLQ) after repeated failing processing. Lambda supports destinations (`SNS`, `SQS`, `Lambda` & `EventBridge`) where successful or failed events can be sent.
- **Event source mappings**. Typically used on streams or queues which don't support event generation to invoke lambda (`Kinesis`, `DynamoDB` streams, `SQS`). Permissions from the lambda execution role are used by the event source mapping to interact with the event source. `SQS` queues or `SNS` topics can be used for any discarded failed event batches.

Lambda functions have versions (v1, v2, v3, ...). A version is the code + the configuration of the lambda function. Version is immutable: it never changes once published and has its own `Amazon Resource Name` (`ARN`). `$Latest` points at the latest version. Aliases (DEV, STAGE, PROD) point at a version and can be changed.

An execution context is the environment a lambda function runs in. A cold start (~100ms) is a full creation and configuration including function code download. With a warm start (~1-2ms), the same execution context is reused. If used infrequently, context will be removed. Concurrent executions will use multiple (potentially new) contexts. You can use provisioned concurrency to improve start speeds: AWS will create and keep X contexts warm and ready to use.

## CloudWatch Events and EventBridge

`EventBridge` is a more modern superset of `CloudWatch Events`.

Both products allow to implement the logic of "if X happens, or at Y time(s) – do Z".

In `CloudWatch Events` there is only one event bus (default event bus for the account; not exposed, used implicitly). `EventBridge` can have additional event buses.

Rules match incoming events or schedules and route the events to 1+ targets (e.g. lambda).

## Serverless

You manage few, if any, servers (low admin overhead). Applications are a collection of small and specialized functions running in stateless and ephemeral environments (with duration billing). Architecture generally is event-driven; compute consumption occurs only when being used. FaaS is used where possible for compute functionality. Managed services are used where possible.

## Simple Notification Service (SNS)

Highly available, regionally resilient & scalable, durable, secure pub/sub messaging service. It is a public AWS service. Coordinates the sending and delivery of messages. Message payload size is limited to 256KB. `SNS Topics` are the base entity of SNS; allow to define permissions and configuration. A `publisher` **sends** messages to a `topic`. `Topics` have `subscribers` which **receive** messages. It is possible to setup a filter on a subscriber, so it will receive only specific messages from the topic.

Types of subscribers:

- HTTP(S)
- Email
- SQS
- Mobile Push
- SMS Messages
- Lambda

SNS used across AWS for notifications (e.g. `CloudWatch`, `CloudFormation`).

Fanout architecture: single SNS Topic, multiple SQS Queues.

SNS supports delivery status (with subscribers like HTTP/Lambda/SQS), delivery retries, server-side encryption (`SSE`), cross-account (via topic policy).

## Step Functions

Allow to create a serverless workflow represented by a state machine. Maximum duration of a state machine execution is **1 year**. Supports two different workflows:

- `standard`: default, **1 year** limit
- `express`: designed for high volume event processing workload, like: IoT, streaming, mobile processing; **5 minutes** limit

Can be started via `API Gateway`, `IoT Rules`, `EventBridge`, `Lambda`. State machines can be configured via Amazon States Language (`ASL`) JSON template. `IAM Role` is used for permissions.

An example of a state machine defined in ASL:

```json
{
  "Comment": "Pet Cuddle-o-Tron - using Lambda for email.",
  "StartAt": "Timer",
  "States": {
    "Timer": {
      "Type": "Wait",
      "SecondsPath": "$.waitSeconds",
      "Next": "Email"
    },
    "Email": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "EMAIL_LAMBDA_ARN",
        "Payload": {
          "Input.$": "$"
        }
      },
      "Next": "NextState"
    },
    "NextState": {
      "Type": "Pass",
      "End": true
    }
  }
}
```

Possible flow-related states:

- `SUCCEED` / `FAIL`: final
- `WAIT`: for duration / until time
- `CHOICE`: condition
- `PARALLEL`: simultaneous processing
- `MAP`: perform some action for each item from a list
- `TASK`: unit of work, which can use:
  - Lambda
  - Batch
  - DynamoDB
  - ECS
  - SNS
  - SQS
  - Glue
  - SageMaker
  - EMR
  - Step Functions

Example of a trust policy for a state machine:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": ["sts:AssumeRole"],
      "Effect": "Allow",
      "Principal": {
        "Service": ["states.amazonaws.com"]
      }
    }
  ]
}
```

An example of permission policies for a state machine:

```json
[
  {
    "PolicyName": "cloudwatchlogs",
    "PolicyDocument": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "logs:CreateLogGroup",
            "logs:CreateLogStream",
            "logs:PutLogEvents",
            "logs:CreateLogDelivery",
            "logs:GetLogDelivery",
            "logs:UpdateLogDelivery",
            "logs:DeleteLogDelivery",
            "logs:ListLogDeliveries",
            "logs:PutResourcePolicy",
            "logs:DescribeResourcePolicies",
            "logs:DescribeLogGroups"
          ],
          "Resource": "*"
        }
      ]
    }
  },
  {
    "PolicyName": "invokelambdasandsendSNS",
    "PolicyDocument": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": ["lambda:InvokeFunction", "sns:*"],
          "Resource": "*"
        }
      ]
    }
  }
]
```

## API Gateway

Used to create and manage APIs. Acts as an endpoint/entry-point for applications. Sits between applications and integrations (services). Highly available, scalable, handles authorization, throttling, caching, CORS, transformations, OpenAPI spec, direct integration and much more. Can connect to services/endpoints in AWS or on-premises.

General API Gateway flow:

- request
  - authorize
  - validate
  - transform
- response
  - transform
  - prepare
  - return

`API Gateway` can use `CloudWatch Logs` to store and manage full stage request and response logs. `CloudWatch` can store metrics for client and integration sides. `API Gateway Cache` can be used to reduce the number of calls made to backend integrations and improve client performance.

`Cognito` or a custom `Lambda` authorizer function can be used for authorization.

Endpoint types:

- `Edge-Optimized`: routed to the nearest CloudFront POP (point-of-presence)
- `Regional`: clients in the same region
- `Private`: endpoint accessible only within a VPC via interface endpoint

APIs are deployed to **stages**; each stage has one deployment; each stage has its own URL and settings.

Stages can be enabled for canary deployments. If done, deployments are made to the canary (sub-part of the stage), not the stage. Stages enabled for canary deployments can be configured so a certain percentage of traffic is sent to the canary. This can be adjusted over time; or the canary can be promoted to make it the new base stage.

Errors:

- `4xx` - client error: invalid request on client side
  - `400` - bad request: generic
  - `403` - access denied: authorizer denies or `WAF` filtered
  - `429` - too many requests: API Gateway can throttle
- `5xx` - server error: valid request, backend issue
  - `502` - bad gateway: bad output returned by lambda
  - `503` - service unavailable: backing endpoint offline, major service issues, etc.
  - `504` - integration failure/timeout: 29s limit

Cache is defined per **stage** within API Gateway. Cache TTL default is 300 seconds. Configurable from 0 to 3600s. Cache can be encrypted. Cache size is from 500MB to 237GB. Cache provides reduced load and cost; it is also improves performance.

## Simple Queue Service (SQS)

SQS is a public, fully managed, highly available service. Provides managed message queues of two types:

- **Standard**: at-least-once. The order is not guaranteed. Multi-lane highway. Performance: can be scaled almost indefinitely. Great for: decoupling application components, worker pools, batch for future processing.
- **FIFO**: exactly-once. Guaranteed order. Single-lane highway. Performance: 3000 messages per second with batching or up to 300 messages per second without. Must have `.fifo` suffix. Great for: workflow ordering, command ordering, price adjustments.

Messages up to 256KB in size. Received messages are hidden for a `VisibilityTimeout` duration, after which they will reappear on the queue unless the client explicitly deleted the messages from the queue. `VisibilityTimeout` is 30s by default and can be set on queue or per-message from 0 to 12 hours. Dead-Letter queues can be used for problem messages. ASGs can scale and Lambdas invoke based on queue length.

In case if single messages requires several parallel processing actions (example: transcoding the same video in different resolutions), the common architectural pattern is to send the message to a specific SNS topic first, and have several SQS queues (per action type) subscribed to this topic. This is called SNS and SQS Fanout.

SQS is billed based on requests. One request is 0-10 messages up to 64KB total.

Different types of polling:

- **short** (immediate): uses one request and can receive 0 (still billable) or more messages
- **long** (`waitTimeSeconds`, up to 20 seconds): if there are messages on the queue, returns them immediately; otherwise waits for messages; better to use

Support encryption at rest (KMS) and in-transit.

Access to queue is based on identity policies or you also can use a queue policy (resource policy).

With a Delay Queue we're configuring the `DelaySeconds`. Messages added to the queue will be **invisible** for this duration. Default value is 0; maximum is 15 minutes. Message timers allow a per-message invisibility to be set (not supported on **FIFO** queues), overriding any queue setting.

`ReceiveCount` is the counter of how many times message been timed out by the `VisibilityTimeout`. You can define the **redrive policy**: specifies the Source Queue, the Dead-Letter Queue and the conditions where messages will be moved from one to the other. Usually by defining `maxReceiveCount` such as when `ReceiveCount` > `maxReceiveCount` and the message isn't deleted, it's moved to the dead-letter queue. Enqueue timestamp of message is unchanged. Retention period of a dead-letter queue is generally longer than the source queue.

## Kinesis Data Streams

Kinesis is a public & highly available scalable streaming service, designed to ingest a lot of data from a large amount of devices or applications. Producers send data into a Kinesis stream. Streams can scale from low to near infinite data rates.

Streams store a `24-hour` (can be increased to a maximum of `365 days` for an additional cost) moving window of data. Multiple consumers access data from that moving window.

Some examples of producers:
- EC2 instance
- on-premise server
- mobile application/device
- IoT sensor

Some examples of consumers:
- on-premise server
- EC2 instance
- Lambda function

Scaling of Kinesis stream is achieved through sharding (additional shards added when load increases). Each shard provides `1MB/s` ingestion capacity and `2MB/s` consumption capacity. More shards means higher price.

Data is stored in Kinesis Data Record (`1MB`).

### SQS vs Kinesis

- SQS
  - commonly used with 1 production group and 1 consumption group
  - used for decoupling and asynchronous communications
  - doesn't provide the persistence of messages and doesn't provide the time window concept
- Kinesis
  - commonly used with multiple consumers
  - designed for huge scale ingestion
  - has a concept of rolling time window
  - often used for:
    - data ingestion
    - analytics
    - monitoring
    - app clicks

## Kinesis Data Firehose

Fully managed service to load data for data lakes, data stores and analytics services. Scales automatically, fully serverless, resilient. **Near** real-time delivery (~60 seconds, lol); significantly slower than Kinesis Data Stream (~200 ms). Data delivered when buffer (1 MB) fills or buffer interval (60 seconds) passes (both values can be adjusted). Supports transformation of data on the fly (lambda). Billing depends on data volume passing through firehose.

Valid destinations for Kinesis Data Firehose:
- HTTP
- Splunk
- Redshift (uses S3 as an intermediate)
- ElasticSearch
- S3 bucket

Can consume data already ingested into Kinesis Data Stream.

## Kinesis Data Analytics

Service which provides real-time processing of data using Structured Query Language (SQL).

Supported sources (ingestion):
- Kinesis Data Streams
- Kinesis Data Firehose

Supported destinations:
- Kinesis Data Firehose (**near** real-time, following up by S3, Redshift, ElasticSearch, Splunk)
- Lambda (real-time)
- Kinesis Data Streams (real-time)

You create Kinesis Analytics Application, which uses sources to form an in-application input stream (optionally enriched by the static data from reference table, backed by S3). Application code processes input and produces output (no way), which is added to an in-application output streams. These streams are fed to the destinations. Errors, generated from processing, are added to an in-application error stream.

Scenarios to use Kinesis Data Analytics:
- streaming data needing real-time SQL processing
- time-series analytics (examples: elections/e-sports)
- real-time dashboards (example: leaderboards for games)
- real-time metrics (example: security & response teams)

## Kinesis Video Streams

Allow to ingest live video data from producers:
- security cameras
- smartphones
- cars
- drones
- time-serialized data
  - audio
  - thermal
  - depth
  - radar

Consumers can access data frame-by-frame or as needed.

Service can persist and encrypt (in-transit and at rest) the data. The data **can't be accessed directly** via storage; only via APIs.

Integrated with other AWS services, e.g. `Rekognition` (example: facial recognition) and `Connect` (example: voice mail).

## Amazon Cognito

Service provides **authentication**, **authorization** and **user management** for web/mobile apps.

Two parts:
- `user pools`
  - allows to sign-up/sign-in (customizable web UI) and get a JSON Web Token (JWT)
  - provides user directory management and profiles
  - supports social identity providers (Google, Facebook, Twitter, etc.)
  - supports MFA and other security features
- `identity pools`
  - allow to offer access to temporary AWS credentials
  - supports unauthenticated identities (guests)
  - supports federated identities (Google, Facebook, Twitter, SAML2.0, user pool) swap for short term AWS Credentials to access AWS resources

General flow with User Pools:
- User (either from an internal pool, or via a social provider) authenticates and receives User Pool Token (JWT)
- User Pool Token used as proof of authentication to access self-managed server-based resources
- User Pool Token can grant access to API via legacy Lambda custom authorizers and now directly via API Gateway

General flow with Identity Pools:
- User authenticated via 3rd-party identity provider (IDP) directly and gets an IDP token
- User passes a token to a a pre-configured (for a specific IDPs) Cognito
- Cognito also pre-configured for two roles: authenticated and unauthenticated
- Cognito assumes an IAM role defined in Identity Pool and returns temporary AWS credentials
- User uses temporary AWS credentials to access AWS resources
- Expired temporary AWS credentials are renewed via Cognito as needed

Combined flow with User & Identity Pools:
- Cognito User Pool pre-configured with social sign-in
- User Pool generates a User Pool Token even if user signed-in with social IDP
- Application passes a User Pool Token to an Identity Pool
- Identity Pool is pre-configured for two roles: authenticated and unauthenticated
- Cognito assumes an IAM role defined in Identity Pool and returns temporary AWS credentials
- Application uses temporary AWS credentials to access AWS resources
- Expired temporary AWS credentials are renewed via Cognito as needed

## AWS Glue

`AWS Glue` is a serverless ETL (Extract, Transform & Load) system. There is a similar service `AWS Data Pipeline`, which can also do ETL and uses servers (EMR). AWS Glue moves and transforms data between a source and a destination. It also crawls data sources and generates the AWS Glue Data catalog.

Possible data sources:
- store
  - S3
  - RDS
  - JDBC DBs
  - DynamoDB
- stream
  - Kinesis Data Stream
  - Apache Kafka

Possible data targets:
- S3
- RDS
- JDBC DBs

AWS Glue Data Catalog is a persistent metadata about data sources in a region. Provides one catalog per region per account. Allows to avoid data silos (?).

Various AWS services are able to use Glue for ETL and Catalog services:
- Amazon Athena
- Redshift Spectrum
- EMR
- AWS Lake Formation

Glue jobs can be initiated manually or via events using EventBridge. When resources are required, Glue allocates from an AWS Warm Pool to perform the ETL processes.

## Amazon MQ

Amazon MQ is an open-source message broker (like a merge of SQS and SNS, but using open standards). Based on Managed Apache ActiveMQ. Supports JMS API and protocols such as AMQP, MQTT, OpenWire, STOMP. Provides both queues and topics (one-to-one or one-to-many). Supports Single Instance (for test / dev environment) or HA Pair (Active / Standby, for production). VPC-based (**not a public**) service, private networking required. No AWS native integration: delivers ActiveMQ product, which you manage.

How to choose between SQS/SNS and Amazon MQ:
- SNS or SQS
  - for most new implementations
  - if AWS integration is required (logging, permissions, encryption, service integration)
- AmazonMQ:
  - if you need to migrate from an existing system with little to no application change
  - id APIs such as JMS or protocols such as AMQP, MQTT, OpenWire or STOMP are needed

## Amazon AppFlow

AppFlow is a fully-managed integration service. Allows to exchange data between applications (connectors) using flows. Flow consists of a source connector and a destination connector.

Usage examples:
- sync data across applications
- aggregate data from different sources

By default the service uses public endpoints, but works with PrivateLink. Supports connectors for many existing applications, and also allows to build a custom connector using AppFlow Custom Connector SDK.

Connections store configuration & credentials to access applications. Connections are defined separately and can be reused across many flows. Within the flow we define:
- source & destination field mapping
- optional data transformation
- optional filters and validation

## Quiz

- How are permissions provided to Lambda functions
  - execution role
- How long can lambda functions run for
  - Max 15 minutes
- Lambda runtime environments are described as?
  - Ephemeral/Stateless
- SQS Queues come in two types (choose two)
  - FIFO
  - Standard
- Which configuration value controls how long something has to process and delete a queue message before it reappears
  - Visibility Timeout
- if you have a large number of devices sending data into AWS to be consumed by a large number of devices you should use a ...
  - Kinesis Stream
- If you are trying to decouple a super high volume application you should use a (choose one)
  - SQS Standard Queue
- What needs to be changed to improve the performance of a kinesis stream
  - Stream Shards
- Which service should be used for serverless long running workflows
  - Step Functions
- Which AWS service supports PUB SUB Messaging
  - SNS
- Which architecture should be used when one event needs to initiate multiple workflow processes
  - SNS + SQS Fanout
- What services are commonly used within serverless architectures
  - API Gateway
  - Lambda
  - Step Functions
  - S3