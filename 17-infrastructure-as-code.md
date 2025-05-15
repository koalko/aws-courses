# Infrastructure as Code (CloudFormation)

## Physical & Logical Resources

CloudFormation **template** (YAML or JSON) contains logical resources ("what to create"). Templates are used to create **stacks**. Job of a stack is to create physical resources based on logical resources, defined within the template.

If a stacks template is changed – physical resources are changed. If a stack is deleted – normally, the physical resources are also deleted.

Resource **properties** are used by CloudFormation when creating the matching physical resources. Once a logical resource moves to `create_complete` (meaning the physical resource is active) it can be queried for attributes of the physical resource within the template.

## Simple Non Portable Template

Example:

```yaml
Resources:
  Bucket:
    Type: "AWS::S3::Bucket"
    Properties:
      BucketName: "top-10-koala-pics"
  Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      InstanceType: "t2.micro"
      ImageId: "ami-090fa75af13c156b4"
```

Issues:

- hard-coded bucket name
- hard-coded image ID

## Template and Pseudo Parameters

Template parameters accept input (via console/CLI/API) when a stack is created or updated. Can be referenced from within logical resources; influence physical resources and/or configuration. Can be configured with `Defaults`, `AllowedValues`, `Min` and `Max` length, `AllowedPatterns`, `NoEcho`, `Type`.

Template parameters example:

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: "t3.micro"
    AllowedValues:
      - "t3.micro"
      - "t3.medium"
      - "t3.large"
    Description: "Pick a supported InstanceType."
  InstanceAmiId:
    Type: String
    Description: "AMI ID For Instances."
```

Pseudo parameters are similar to template parameters, but they are provided by AWS based on the environment when creating the stack. Examples:

- `AWS::Region`
- `AWS::StackId`
- `AWS::StackName`
- `AWS::AccountId`

## Intrinsic Functions

- `Ref` & Fn::`GetAtt`: reference a value from one logical resource or a parameter in another one
- Fn::`Join`: join strings together
- Fn::`Split`: split strings up
- Fn::`GetAZs`: get a list of AZs in a given region
- Fn::`Select`: select one element from the list
- Conditions:
  - Fn::`If`
  - Fn::`And`
  - Fn::`Equals`
  - Fn::`Not`
  - Fn::`Or`
- Fn::`Base64`: encode to base64
- Fn::`Sub`: substitute values in text
- Fn::`Cidr`: build CIDR
- Some others:
  - Fn::`ImportValue`
  - Fn::`FindInMap`
  - Fn::`Transform`

`Ref` example:

```yaml
Resources:
  Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: "t3.micro"
      KeyName: "A4L"
```

Using `!Ref` on template or pseudo parameters returns their value. When used with logical resources, the physical ID is usually returned.

`!GetAtt` can be used to retrieve any attribute associated with the resource. Most logical resources return detailed configuration of the physical resource.

Use `!GetAZs "us-east-1"` to get AZs in a specific region (`us-east-1` in that example) or `!GetAZs ""` to get AZs in the current region. Example of using `!GetAZs` together with `!Select`:

```yaml
Instance:
  Type: "AWS::EC2::Instance"
  Properties:
    ImageId: !Ref LatestAmiId
    InstanceType: "t3.micro"
    KeyName: "A4L"
    AvailabilityZone: !Select [0, !GetAZs ""]
```

`!Split[ "|", "roffle|truffles|penny|winkie" ]` will return `[ "roffle", "truffles", "penny", "winkie" ]`.

`!Join[ "|", [ "roffle", "truffles", "penny", "winkie" ] ]` will return the initial string.

Realworld `!Join` example:

```yaml
WordpressURL:
  Description: Instance Web URL
  Value: !Join ["", ["http://", !GetAtt Instance.DNSName]]
```

Fn::`Base64` and Fn::`Sub` example:

```yaml
UserData:
  Fn::Base64: !Sub |
    #!/bin/bash -xe
    yum -y update
    yum -y install httpd
    amazon-linux-extras install -y php7.2
    amazon-linux-extras install epel -y
    echo "<html><head></head><body style=\"background-color:#$bgcolor;\">" >> /var/www/html/index.html
    echo "<center><h1>Instance ID: ${Instance.InstanceId}</h1></center><br>" >> /var/www/html/index.html
    echo "<center><img src=\"cat.gif\"></center>" >> /var/www/html/index.html
    echo "</body></html>" >> /var/www/html/index.html
```

This is actually an invalid example, because `Instance.InstanceId` tries to reference itself.

Fn::`Cidr` is used to generate a number of smaller CIDR ranges for subnets from a larger VPC range. Example:

```yaml
VPC:
  Type: AWS::EC2::VPC
  Properties:
    CidrBlock: "10.16.0.0/16"
Subnet1:
  Type: AWS::EC2::Subnet
  Properties:
    CidrBlock: !Select ["0", !Cidr [!GetAtt VPC.CidrBlock, "16", "12"]]
    VpcId: !Ref VPC
Subnet2:
  Type: AWS::EC2::Subnet
  Properties:
    CidrBlock: !Select ["1", !Cidr [!GetAtt VPC.CidrBlock, "16", "12"]]
    VpcId: !Ref VPC
```

## Mappings

Templates can contain a Mappings object, which can contain many mappings, each map keys to values, allowing lookup. Can have one key, or top & second level keys. Mappings use the `!FindInMap` intrinsic function (format: `!FindInMap [ MapName, TopLevelKey, SecondLevelKey ]`).

Common use: retrieve AMI for a given region and architecture. Improve template portability.

Example:

```yaml
Mappings:
  RegionMap:
    us-east-1:
      HVM64: "ami-Off8a91507f77f867"
      HVMG2: "ami-0a584ac55a7631c0c"
    us-west-1:
      HVM64: "ami-0bdb828fd58c52235"
      HVMG2: "ami-066ee5fd4a9ef77f1"
    eu-west-1:
      HVM64: "ami-047bb4163c506cd98"
      HVMG2: "ami-31c2f645"
```

To find AMI in such mapping:

```yaml
!FindInMap ["RegionMap", !Ref "AWS:Region", "HVM64"]
```

## Outputs

Templates can have an optional Outputs section. Values, which are declared in this section:

- visible as outputs when using the CLI
- visible as outputs in the console UI
- accessible from a parent stack when using nesting
- can be exported, allowing cross-stack references

An example:

```yaml
Outputs:
  WordpressURL:
    Description: "Instance Web URL"
    Value: !Join ["", ["https://", !GetAtt Instance.DNSName]]
```

Description is visible from the CLI and Console UI & passed back to parent stack when nested stacks are used.

## Template portability

Non-portable template (hard-coded bucket name and AMI ID):

```yaml
Resources:
  Bucket:
    Type: "AWS::S3::Bucket"
    Properties:
      BucketName: "accatpics1333333337"
  Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      InstanceType: "t2.micro"
      ImageId: "ami-090fa75af13c156b4"
```

To auto-generate a bucket name, just omit the `BucketName` parameter:

```yaml
Resources:
  Bucket:
    Type: "AWS::S3::Bucket"
  Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      InstanceType: "t2.micro"
      ImageId: "ami-090fa75af13c156b4"
```

To move the AMI ID to parameters:

```yaml
Parameters:
  AMIID:
    Type: "String"
    Description: "AMI for EC2"
Resources:
  Bucket:
    Type: "AWS::S3::Bucket"
  Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      InstanceType: "t2.micro"
      ImageId: !Ref "AMIID"
```

To provide a proper type for AMI ID, and pre-populate it with automatically retrieved image ID from desired region:

```yaml
Parameters:
  LatestAmiId:
    Description: "AMI for EC2"
    Type: "AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>"
    Default: "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
Resources:
  Bucket:
    Type: "AWS::S3::Bucket"
  Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      InstanceType: "t2.micro"
      ImageId: !Ref "LatestAmiId"
```

## Conditions

Created in the optional "Conditions" section of a template. Evaluated to `TRUE` or `FALSE`. Processed before resources are created. Use other intrinsic functions: AND, EQUALS, IF, NOT, OR. Associated with logical resources to control if they are created or not.

Example:

```yaml
Parameters:
  EnvType:
    Default: "dev"
    Type: String
    AllowedValues:
      - "dev"
      - "prod"
```

...

```yaml
Conditions:
  IsProd: !Equals
    - !Ref EnvType
    - "prod"
```

...

```yaml
Resources:
  Wordpress:
    Type: "AWS::EC2::Instance"
    Condition: IsProd
    Properties:
      ImageId: "ami-123456789098"
```

## DependsOn

For efficiency CloudFormation does things (create, update, delete) in parallel, but tries to determine a dependency order (for example: `VPC` -> `SUBNET` -> `EC2`) using information from references or functions. `DependsOn` lets you explicitly define dependency order.

One example of when implicit dependency detection doesn't work is an Elastic IP, which requires an IGW attached to a VPC in order to work – but there is no dependency in the template.

Example:

```yaml
VPC:
  Type: AWS::EC2::VPC
  Properties:
    CidrBlock: 10.16.0.0/16
    EnableDnsSupport: true
    EnableDnsHostnames: true
    Tags:
      - Key: Name
        Value: a4l-vpc1
```

...

```yaml
InternetGateway:
  Type: "AWS::EC2::InternetGateway"
  Properties:
    Tags:
      - Key: Name
        Value: A4L-vpc1-igw
```

...

```yaml
InternetGatewayAttachment:
  Type: "AWS::EC2::VPCGatewayAttachment"
  Properties:
    VpcId: !Ref VPC
    InternetGatewayId: !Ref InternetGateway
```

...

```yaml
WPEIP:
  Type: AWS::EC2::EIP
  DependsOn: InternetGatewayAttachment
  Properties:
    InstanceId: !Ref WordpressEC2
```

## Wait Conditions & cfn-signal

You configure CloudFormation to **hold** and wait for `X` number of **success** signals within the **timeout** (`H:M:S`, 12 hour max). The resource state will change to `CREATE_COMPLETE` only if the expected number of **success** signals is received within the **timeout** period. If **failure** signal received or if **timeout** is reached – creation fails.

With EC2 or Auto Scaling Group you should use `CreationPolicy`, but for more general cases there is a `WaitCondition` logical resource.

An example of `CreationPolicy` section of an `AutoScalingGroup` resource:

```yaml
CreationPolicy:
  ResourceSignal:
    Count: "3"
    Timeout: PT15M
```

Using `cfn-signal` in `UserData` (`AWS::AutoScaling::LaunchConfiguration`):

```yaml
/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource AutoScalingGroup --region ${AWS::Region}
```

`WaitCondition` can depend on other resources; other resources can depend on `WaitCondition`. An example of `WaitCondition`:

```yaml
WaitCondition:
  Type: AWS::CloudFormation::WaitCondition
  DependsOn: "someresource"
  Properties:
    Handle: !Ref "WaitHandle"
    Timeout: "300"
    Count: "1"
```

`WaitHandle` generates a pre-signed URL for resource signals.

```yaml
WaitHandle:
  Type: AWS::CloudFormation::WaitConditionHandle
```

Accessing `WaitCondition` data: `!GetAtt WaitCondition.Data`.

## Nested Stacks

Resources in a single stack share a lifecycle.

Stack limits:

- resources number (500)
- can't easily reuse resources (e.g. VPC)
- can't easily reference other stacks

With nested stacks you start with the `Root` Stack. `Parent` Stack is a parent of any stack which it immediately creates (`Root` Stack is also a `Parent` Stack).

Stack as a logical resource:

```yaml
VPCSTACK:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://someurl.com/template.yaml
    Parameters:
      Param1: !Ref SomeParam1
      Param2: !Ref SomeParam2
      Param3: !Ref SomeParam3
```

Nested stack outputs can be referenced as `VPCSTACK.Outputs.XXXXX`. One stack can depend on another via `DependsOn`.

Whole templates can be reused in other stacks.

Use nested stacks:

- when the stacks form part of one solution – lifecycle linked
- to overcome the 500 resource limit of one stack
- to modularize templates (code reuse)
- to make the installation process easier (nested stacks created by the root stack)
- only when everything is lifecycle linked

## Cross-Stack References

CFN stacks are designed to be **isolated** and **self-contained**. Outputs are normally not visible from other stacks (exception: parent/nested stacks). Outputs can be **exported**, making them visible from other stacks. Exports must have a unique name in the region. Exports are visible within the account + the region. `Fn::ImportValue` can be used instead of `Ref`.

Example for a shared VPC Stack:

```yaml
Outputs:
  SHAREDVPCID:
    Description: Shared Services VPC
    Value: !Ref VPC
    Export:
      Name: SHAREDVPCID
```

Using the exported variable within the same account/region: `!ImportValue SharedVPCID`.

It makes sense to use Cross-Stack References:

- for service-oriented architectures (provide services from one stack to another)
- for resources with different lifecycles
- for stack reuse

Important note: `nested stacks` allow **template** reuse; `cross-stack references` allow **stack** reuse.

## Stack Sets

Allow to deploy CFN stacks across many accounts & regions. `StackSets` are containers in an **admin account**. `StackSet` contains stack instances, which reference stacks. Stack instances and stacks are created in **target accounts**. Each stack is created in 1 region / 1 account. Self-managed (Organization) roles or service-managed (IAM) roles are used for creating stacks.

The template for the stack set is a normal CFN template.

Options to set when creating a `StackSet`:

- Concurrent Accounts: defines how many AWS accounts can be used at the same time
- Failure Tolerance: amount of individual deployments which can fail before the stack itself will be viewed as failed
- Retain Stacks: by default, removing the stack instance from the stack set will also remove the created stack; this option allow to alter this behaviour

Example scenarios:

- Enable AWS Config
- AWS Config Rules: MFA, EIPS, EBS Encryption
- Create IAM Roles for cross-account access

## Deletion Policy

If you delete a logical resource from a template, by default the physical resource is also deleted. This can cause data loss in some cases (RDS, EC2+EBS, etc.). You can define a deletion policy on each resource:

- `Delete` (default)
- `Retain`
- `Snapshot` (supported for EBS Volume, ElastiCache, Neptune, RDS, Redshift)

Snapshots continue on past stack lifetime.

Deletion policy only applies to **delete**, **not replace** (you can still experience data loss, for example, if replacing RDS instance with slightly different configuration, even with `Retain` deletion policy).

## Stack Roles

CFN uses the permissions of the logged in identity, which means you need permissions to interact with stacks and to manipulate AWS resources. CFN can assume a role to gain the permissions. This lets you implement role separation. The identity creating the stack doesn't need resource permissions: only stack permissions and `PassRole`.

Example:

- Account Admin creates IAM Role with permissions to create, update and delete AWS resources
- Account HelpDesk has no permissions to assume the role or edit it
- Account HelpDesk only has permissions to manipulate CFN stacks, and to `pass` the IAM Role into CFN
- IAM Role is attached to the stack and used for any operations

## `AWS::CloudFormation::Init` and `cfn-init`

Simple configuration management system. Configuration directives stored in template under `AWS::CloudFormation::Init` part of logical resource.

User data is procedural (**how**). `cfn-init` is a desired state (**what**).

`cfn-init` is a helper script, which is installed on EC2 OS.

Template example:

```yaml
EC2Instance:
  Type: AWS::EC2::Instance
  CreationPolicy: ...
  Metadata:
    AWS::CloudFormation::Init:
      configSets: ...
      install_cfn: ...
      software_install: ...
      configure_instance: ...
      install_wordpress: ...
      configure_wordpress: ...
```

Usage from `UserData`:

```yaml
UserData:
  Fn::Base64: !Sub |
    #!/bin/bash -xe
    yum -y update
    /opt/aws/bin/cfn-init -v --stack ${AWS::StackId} --resource EC2Instance --configsets wordpress_install --region ${AWS::Region}
    /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource EC2Instance --region ${AWS::Region}
```

ConfigKeys (`install_cfn`, `software_install`, and others from example above) are containers for configuration with following sections:

```yaml
packages:
  # packages to install
groups:
  # local group mgmt
users:
  # local user mgmt
sources:
  # download and extract archives
files:
  # files to create
commands:
  # commands to execute
services:
  # services to enable
```

ConfigSets define which ConfigKeys to use and in which order to apply.

## `cfn-hup`

`cfn-init` is run once as part of bootstrapping (used data); if `CloudFormation::Init` is updated, it **isn't rerun**.

`cfn-hup` helper is a daemon which can be installed on EC2 instance; it detects changes in resource metadata, running configurable actions when a change is detected. This, for example, allow to update config on EC2 instances when the stack is updated.

Example:

- User changes the CFN template
- UpdateStack operation is performed
- `cfn-hup` checks metadata periodically
- when metadata updated, `cfn-hup` calls `cfn-init`
- `cfn-init` applies new configuration

## `cfn-signal`, `cfn-init` and `cfn-hup` examples

### Template with simulated delay

```yaml
Parameters:
  LatestAmiId:
    Description: "AMI for EC2"
    Type: "AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>"
    Default: "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
  Message:
    Description: "Message for HTML page"
    Default: "Cats are the best"
    Type: "String"
Resources:
  InstanceSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupDescription: Enable SSH and HTTP access via port 22 IPv4 & port 80 IPv4
      SecurityGroupIngress:
        - Description: "Allow SSH IPv4 IN"
          IpProtocol: tcp
          FromPort: "22"
          ToPort: "22"
          CidrIp: "0.0.0.0/0"
        - Description: "Allow HTTP IPv4 IN"
          IpProtocol: tcp
          FromPort: "80"
          ToPort: "80"
          CidrIp: "0.0.0.0/0"
  Bucket:
    Type: "AWS::S3::Bucket"
  Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      InstanceType: "t2.micro"
      ImageId: !Ref "LatestAmiId"
      SecurityGroupIds:
        - !Ref InstanceSecurityGroup
      Tags:
        - Key: Name
          Value: A4L-UserData Test
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          yum -y update
          yum -y upgrade
          # simulate some other processes here
          sleep 300
          # Continue
          yum install -y httpd
          systemctl enable httpd
          systemctl start httpd
          echo "<html><head><title>Amazing test page</title></head><body><h1><center>${Message}</center></h1></body></html>" > /var/www/html/index.html
```

Problems:

- disconnect between the bootstrapping process and when the stack is completes
- instance configuration (`UserData`) isn't updated when the stack is updated

### Template with `cfn-signal`

```yaml
Parameters:
  LatestAmiId:
    Description: "AMI for EC2"
    Type: "AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>"
    Default: "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
  Message:
    Description: "Message for HTML page"
    Default: "Cats are the best"
    Type: "String"
Resources:
  InstanceSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupDescription: Enable SSH and HTTP access via port 22 IPv4 & port 80 IPv4
      SecurityGroupIngress:
        - Description: "Allow SSH IPv4 IN"
          IpProtocol: tcp
          FromPort: "22"
          ToPort: "22"
          CidrIp: "0.0.0.0/0"
        - Description: "Allow HTTP IPv4 IN"
          IpProtocol: tcp
          FromPort: "80"
          ToPort: "80"
          CidrIp: "0.0.0.0/0"
  Bucket:
    Type: "AWS::S3::Bucket"
  Instance:
    Type: "AWS::EC2::Instance"
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M
    Properties:
      InstanceType: "t2.micro"
      ImageId: !Ref "LatestAmiId"
      SecurityGroupIds:
        - !Ref InstanceSecurityGroup
      Tags:
        - Key: Name
          Value: A4L-UserData Test
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          yum -y update
          yum -y upgrade
          # simulate some other processes here
          sleep 300
          # Continue
          yum install -y httpd
          systemctl enable httpd
          systemctl start httpd
          echo "<html><head><title>Amazing test page</title></head><body><h1><center>${Message}</center></h1></body></html>" > /var/www/html/index.html
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource Instance --region ${AWS::Region}
```

Combination of `CreationPolicy/ResourceSignal` and `cfn-signal` allows for stack to move to a CREATE_COMPLETE state only after the bootstrapping process has finished.

### Template with `cfn-init` and `cfn-signal`

```yaml
Parameters:
  LatestAmiId:
    Description: "AMI for EC2"
    Type: "AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>"
    Default: "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
  Message:
    Description: "Message for HTML page"
    Default: "Cats are the best"
    Type: "String"
Resources:
  InstanceSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupDescription: Enable SSH and HTTP access via port 22 IPv4 & port 80 IPv4
      SecurityGroupIngress:
        - Description: "Allow SSH IPv4 IN"
          IpProtocol: tcp
          FromPort: "22"
          ToPort: "22"
          CidrIp: "0.0.0.0/0"
        - Description: "Allow HTTP IPv4 IN"
          IpProtocol: tcp
          FromPort: "80"
          ToPort: "80"
          CidrIp: "0.0.0.0/0"
  Bucket:
    Type: "AWS::S3::Bucket"
  Instance:
    Type: "AWS::EC2::Instance"
    Metadata:
      "AWS::CloudFormation::Init":
        config:
          packages:
            yum:
              httpd: []
          files:
            /var/www/html/index.html:
              content: !Sub |
                <html><head><title>Amazing test page</title></head><body><h1><center>${Message}</center></h1></body></html>
          commands:
            simulatebootstrap:
              command: "sleep 300"
          services:
            sysvinit:
              httpd:
                enabled: "true"
                ensureRunning: "true"
                files:
                  - "/var/www/html/index.html"
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M
    Properties:
      InstanceType: "t2.micro"
      ImageId: !Ref "LatestAmiId"
      SecurityGroupIds:
        - !Ref InstanceSecurityGroup
      Tags:
        - Key: Name
          Value: A4L-UserData Test
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          /opt/aws/bin/cfn-init -v --stack ${AWS::StackId} --resource Instance --region ${AWS::Region}
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource Instance --region ${AWS::Region}
```

Note the `Instance/Metadata/AWS::CloudFormation::Init` section and the changes in `UserData` section.

Log files (on EC2 instance) which can help to debug a potential issue:

- `/var/log/cloud-init-output.log`
- `/var/log/cfn-init-cmd.log`
- `/var/log/cfn-init.log`

### Template with `cfn-init`, `cfn-signal` and `cfn-hup`

```yaml
Parameters:
  LatestAmiId:
    Description: "AMI for EC2"
    Type: "AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>"
    Default: "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
  Message:
    Description: "Message for HTML page"
    Default: "Cats are the best"
    Type: "String"
Resources:
  InstanceSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupDescription: Enable SSH and HTTP access via port 22 IPv4 & port 80 IPv4
      SecurityGroupIngress:
        - Description: "Allow SSH IPv4 IN"
          IpProtocol: tcp
          FromPort: "22"
          ToPort: "22"
          CidrIp: "0.0.0.0/0"
        - Description: "Allow HTTP IPv4 IN"
          IpProtocol: tcp
          FromPort: "80"
          ToPort: "80"
          CidrIp: "0.0.0.0/0"
  Bucket:
    Type: "AWS::S3::Bucket"
  Instance:
    Type: "AWS::EC2::Instance"
    Metadata:
      "AWS::CloudFormation::Init":
        config:
          packages:
            yum:
              httpd: []
          files:
            /etc/cfn/cfn-hup.conf:
              content: !Sub |
                [main]
                stack=${AWS::StackName}
                region=${AWS::Region}
                interval=1
                verbose=true
              mode: "000400"
              owner: "root"
              group: "root"
            /etc/cfn/hooks.d/cfn-auto-reloader.conf:
              content: !Sub |
                [cfn-auto-reloader-hook]
                triggers=post.update
                path=Resources.Instance.Metadata.AWS::CloudFormation::Init
                action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackId} --resource Instance --region ${AWS::Region}
                runas=root
              mode: "000400"
              owner: "root"
              group: "root"
            /var/www/html/index.html:
              content: !Sub |
                <html><head><title>Amazing test page</title></head><body><h1><center>${Message}</center></h1></body></html>
          commands:
            simulatebootstrap:
              command: "sleep 300"
          services:
            sysvinit:
              cfn-hup:
                enabled: "true"
                ensureRunning: "true"
                files:
                  - /etc/cfn/cfn-hup.conf
                  - /etc/cfn/hooks.d/cfn-auto-reloader.conf
              httpd:
                enabled: "true"
                ensureRunning: "true"
                files:
                  - "/var/www/html/index.html"
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M
    Properties:
      InstanceType: "t2.micro"
      ImageId: !Ref "LatestAmiId"
      SecurityGroupIds:
        - !Ref InstanceSecurityGroup
      Tags:
        - Key: Name
          Value: A4L-UserData Test
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          /opt/aws/bin/cfn-init -v --stack ${AWS::StackId} --resource Instance --region ${AWS::Region}
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource Instance --region ${AWS::Region}
```

Note the `/etc/cfn/cfn-hup.conf` and `/etc/cfn/hooks.d/cfn-auto-reloader.conf`.

This template allow for instance configuration (`UserData`) to auto-update when the stack is updated.

Log files (on EC2 instance) for `cfn-hup`:

- `/var/log/cfn-hup.log`

## Change sets

Change sets let you preview changes. You can preview multiple different versions. Chosen changes can be applied by executing the change set.

## Custom Resources

Custom resources let CFN integrate with anything it doesn't natively support. To make changes (create/update/remove) in the custom resource, CloudFormation sends the event data to endpoint you define in that custom resource. In case of success, the returned data will be accessible for the stack. Custom resource can respond via the ResponseURL (received with the CFN event).

### Example without the custom resource

```yaml
Description: Basic S3 Bucket
Resources:
  animalpics:
    Type: AWS::S3::Bucket
```

- template defines an S3 bucket
- stack creates a physical S3 bucket
- stack creation is completed
- user manually uploads files to S3 bucket
- admin tries to remove the stack
- error: bucket is not empty

### Example with the custom resource

```yaml
Description: S3 Bucket Using a custom Resource
Resources:
  animalpics:
    Type: AWS::S3::Bucket
  copyanimalpics:
    Type: "Custom::S3Objects"
    Properties:
      ServiceToken: !GetAtt CopyS3ObjectsFunction.Arn
      SourceBucket: "cl-randomstuffforlessons"
      SourcePrefix: "customresource"
      Bucket: !Ref animalpics
  S3CopyRole:
    Type: AWS::IAM::Role
    Properties:
      Path: /
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: S3Access
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Sid: AllowLogging
                Effect: Allow
                Action:
                  - "logs:CreateLogGroup"
                  - "logs:CreateLogStream"
                  - "logs:PutLogEvents"
                Resource: "*"
              - Sid: ReadFromLCBucket
                Effect: Allow
                Action:
                  - "s3:ListBucket"
                  - "s3:GetObject"
                Resource:
                  - !Sub "arn:aws:s3:::cl-randomstuffforlessons"
                  - !Sub "arn:aws:s3:::cl-randomstuffforlessons/*"
              - Sid: WriteToStudentBuckets
                Effect: Allow
                Action:
                  - "s3:ListBucket"
                  - "s3:GetObject"
                  - "s3:PutObject"
                  - "s3:PutObjectAcl"
                  - "s3:PutObjectVersionAcl"
                  - "s3:DeleteObject"
                  - "s3:DeleteObjectVersion"
                  - "s3:CopyObject"
                Resource:
                  - !Sub "arn:aws:s3:::${animalpics}"
                  - !Sub "arn:aws:s3:::${animalpics}/*"
  CopyS3ObjectsFunction:
    Type: AWS::Lambda::Function
    Properties:
      Description: Copies objects into buckets
      Handler: index.handler
      Runtime: python3.9
      Role: !GetAtt S3CopyRole.Arn
      Timeout: 120
      Code:
        ZipFile: |
          import os 
          import json
          import cfnresponse
          import boto3
          import logging

          from botocore.exceptions import ClientError
          client = boto3.client('s3')
          logger = logging.getLogger()
          logger.setLevel(logging.INFO)

          def handler(event, context):
            logger.info("Received event: %s" % json.dumps(event))
            source_bucket = event['ResourceProperties']['SourceBucket']
            source_prefix = event['ResourceProperties'].get('SourcePrefix') or ''
            bucket = event['ResourceProperties']['Bucket']
            prefix = event['ResourceProperties'].get('Prefix') or ''

            result = cfnresponse.SUCCESS

            try:
              if event['RequestType'] == 'Create' or event['RequestType'] == 'Update':
                result = copy_objects(source_bucket, source_prefix, bucket, prefix)
              elif event['RequestType'] == 'Delete':
                result = delete_objects(bucket, prefix)
            except ClientError as e:
              logger.error('Error: %s', e)
              result = cfnresponse.FAILED

            cfnresponse.send(event, context, result, {})

          def copy_objects(source_bucket, source_prefix, bucket, prefix):
            paginator = client.get_paginator('list_objects_v2')
            page_iterator = paginator.paginate(Bucket=source_bucket, Prefix=source_prefix)
            for key in {x['Key'] for page in page_iterator for x in page['Contents']}:
              dest_key = os.path.join(prefix, os.path.relpath(key, source_prefix))
              if not key.endswith('/'):
                print('copy {} to {}'.format(key, dest_key))
                client.copy_object(CopySource={'Bucket': source_bucket, 'Key': key}, Bucket=bucket, Key = dest_key)
            return cfnresponse.SUCCESS

          def delete_objects(bucket, prefix):
            paginator = client.get_paginator('list_objects_v2')
            page_iterator = paginator.paginate(Bucket=bucket, Prefix=prefix)
            objects = [{'Key': x['Key']} for page in page_iterator for x in page['Contents']]
            client.delete_objects(Bucket=bucket, Delete={'Objects': objects})
            return cfnresponse.SUCCESS
```

- template defines an S3 bucket and a custom resource backed by the Lambda
- stack creates a physical S3 bucket
- stack executes Lambda with "create" event
- Lambda puts some files into the S3 bucket
- stack creation is completed
- user manually uploads some more files to S3 bucket
- admin tries to remove the stack
- CFN "knows" that Lambda depends on S3 (from creation process), so it executes Lambda with "delete" event
- Lambda removes **all** the files from the S3 bucket and reports success
- stack removes the S3 bucket
- stack deletion is completed
