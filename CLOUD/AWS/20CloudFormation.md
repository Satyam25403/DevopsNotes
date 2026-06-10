# AWS CloudFormation (Infrastructure as Code)

AWS CloudFormation lets you define and provision your entire AWS infrastructure using code — **YAML or JSON templates**. You describe what you want; CloudFormation builds it, tracks it, and can tear it all down with one command.

> CloudFormation is to AWS what Terraform is to multi-cloud — both are Infrastructure as Code (IaC) tools that automate resource provisioning from configuration files.

---

## Why CloudFormation?

**Without IaC:** Click through the console to create a VPC, subnets, security groups, ECS cluster, task definition, service, IAM roles… 20+ manual steps, error-prone, hard to reproduce.

**With CloudFormation:** Write one YAML file → run one command → everything is provisioned in the correct order, every time.

**Key benefits:**
- **Repeatable**: Same template deploys identically to dev, staging, and prod
- **Version controlled**: Track infrastructure changes in Git like code
- **Auditable**: CloudFormation tracks every change to your stack
- **Rollback**: Automatically reverts if a deployment fails
- **Free**: You pay only for the resources CloudFormation provisions, not CloudFormation itself

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Template** | The YAML/JSON file describing your resources |
| **Stack** | A deployed instance of a template — CloudFormation manages all resources in a stack together |
| **Change Set** | A preview of what CloudFormation will change before applying it |
| **Drift** | When a resource is manually modified outside CloudFormation — detection tells you what drifted |
| **Nested Stack** | A stack that references another stack — for modular, reusable templates |

---

## Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'      # always this value
Description: What this template does

Parameters:                                  # user inputs at deploy time
  ...

Mappings:                                    # static lookup tables (e.g., region-to-AMI)
  ...

Conditions:                                  # conditional resource creation
  ...

Resources:                                   # required — the actual AWS resources
  ...

Outputs:                                     # values to expose after deployment
  ...
```

Only **Resources** is required — everything else is optional.

---

## Parameters (dynamic inputs)

Make templates reusable by accepting values at deploy time:

```yaml
Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
    Description: Deployment environment

  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
    Description: EC2 instance type

  KeyPairName:
    Type: AWS::EC2::KeyPair::KeyName     # validates the key pair exists
    Description: Name of existing EC2 key pair

  DBPassword:
    Type: String
    NoEcho: true                          # hides value in console and logs
    MinLength: 8
    Description: RDS master password

Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType     # reference a parameter
      KeyName: !Ref KeyPairName
```

---

## Intrinsic Functions

CloudFormation provides built-in functions for dynamic values:

| Function | Description | Example |
|----------|-------------|---------|
| `!Ref` | Reference a parameter or resource | `!Ref MyBucket` |
| `!GetAtt` | Get an attribute of a resource | `!GetAtt MyBucket.Arn` |
| `!Sub` | String substitution | `!Sub "arn:aws:s3:::${BucketName}/*"` |
| `!Join` | Join strings with a delimiter | `!Join [",", [a, b, c]]` |
| `!Select` | Select from a list | `!Select [0, !GetAZs ""]` |
| `!GetAZs` | Get AZs in a region | `!GetAZs us-east-1` |
| `!If` | Conditional value | `!If [IsProd, r6i.large, t3.micro]` |
| `!ImportValue` | Import output from another stack | `!ImportValue SharedVpcId` |

---

## Outputs (expose values after deployment)

```yaml
Outputs:
  BucketName:
    Description: Name of the created S3 bucket
    Value: !Ref MyS3Bucket
    Export:
      Name: !Sub "${AWS::StackName}-BucketName"    # exportable to other stacks

  BucketArn:
    Description: ARN of the S3 bucket
    Value: !GetAtt MyS3Bucket.Arn

  WebsiteURL:
    Description: Static website URL
    Value: !GetAtt MyS3Bucket.WebsiteURL

  EC2PublicIP:
    Description: Public IP of the EC2 instance
    Value: !GetAtt WebAppInstance.PublicIp

  EC2WebURL:
    Description: Access the web app here
    Value: !Sub "http://${WebAppInstance.PublicDnsName}"
```

---

## Conditions (conditional resource creation)

```yaml
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, prod]

Conditions:
  IsProduction: !Equals [!Ref Environment, prod]
  IsNotProduction: !Not [!Condition IsProduction]

Resources:
  # Only create a read replica in production
  DBReadReplica:
    Type: AWS::RDS::DBInstance
    Condition: IsProduction
    Properties:
      ...

  # Use larger instance in prod, smaller in dev
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !If [IsProduction, r6i.large, t3.micro]
```

---

## Mappings (static lookup tables)

```yaml
Mappings:
  RegionToAMI:
    us-east-1:
      Amazon2: ami-0c94855ba95c71c99
      Ubuntu: ami-0149b2da6ceec4bb0
    us-west-2:
      Amazon2: ami-01ce4793a2f45922e
      Ubuntu: ami-03c7c1f17ee073747
    eu-west-1:
      Amazon2: ami-0943382e114f188e8
      Ubuntu: ami-0943382e114f188e8

Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionToAMI, !Ref AWS::Region, Amazon2]
```

---

## Example: Static Website on S3

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Static website hosted on S3

Parameters:
  BucketName:
    Type: String
    Description: Globally unique bucket name

Resources:
  StaticSiteBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: error.html
      PublicAccessBlockConfiguration:
        BlockPublicAcls: false
        BlockPublicPolicy: false
        IgnorePublicAcls: false
        RestrictPublicBuckets: false

  BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref StaticSiteBucket
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal: "*"
            Action: s3:GetObject
            Resource: !Sub "${StaticSiteBucket.Arn}/*"

Outputs:
  WebsiteURL:
    Description: S3 static website URL
    Value: !GetAtt StaticSiteBucket.WebsiteURL
```

---

## Example: EC2 Web App with UserData

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Deploy a Node.js app to EC2

Parameters:
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, prod]

Conditions:
  IsProduction: !Equals [!Ref Environment, prod]

Resources:
  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP and SSH
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 3000
          ToPort: 3000
          CidrIp: 0.0.0.0/0

  WebAppInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !If [IsProduction, t3.small, t3.micro]
      KeyName: !Ref KeyName
      ImageId: !FindInMap [RegionToAMI, !Ref AWS::Region, Ubuntu]
      SecurityGroups: [!Ref WebSecurityGroup]
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          apt-get update -y
          apt-get install -y nginx nodejs npm
          systemctl start nginx
          systemctl enable nginx
          echo "<h1>Hello from CloudFormation!</h1>" > /var/www/html/index.html

Mappings:
  RegionToAMI:
    us-east-1:
      Ubuntu: ami-0149b2da6ceec4bb0
    us-west-2:
      Ubuntu: ami-03c7c1f17ee073747

Outputs:
  PublicIP:
    Value: !GetAtt WebAppInstance.PublicIp
  AppURL:
    Value: !Sub "http://${WebAppInstance.PublicDnsName}"
```

---

## Example: Full ECS Fargate Stack

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: ECS Fargate deployment with VPC, IAM, ECR

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: true

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs ""]
      MapPublicIpOnLaunch: true

  ECSSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP to ECS tasks
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  ECSExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

  MyECRRepo:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: my-app-repo

  MyECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: my-cluster

  MyTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      RequiresCompatibilities: [FARGATE]
      Cpu: 256
      Memory: 512
      NetworkMode: awsvpc
      ExecutionRoleArn: !GetAtt ECSExecutionRole.Arn
      ContainerDefinitions:
        - Name: my-app
          Image: !Sub "${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/my-app-repo:latest"
          PortMappings:
            - ContainerPort: 80
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Sub "/ecs/${AWS::StackName}"
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: ecs

  MyECSService:
    Type: AWS::ECS::Service
    Properties:
      Cluster: !Ref MyECSCluster
      DesiredCount: 1
      LaunchType: FARGATE
      TaskDefinition: !Ref MyTaskDefinition
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets: [!Ref PublicSubnet1, !Ref PublicSubnet2]
          SecurityGroups: [!Ref ECSSecurityGroup]
          AssignPublicIp: ENABLED

Outputs:
  ClusterName:
    Value: !Ref MyECSCluster
    Export:
      Name: !Sub "${AWS::StackName}-ClusterName"
  ECRRepoURI:
    Value: !GetAtt MyECRRepo.RepositoryUri
```

---

## Deploying Stacks

### Via Console
1. **CloudFormation → Create Stack → With new resources**
2. Upload your YAML file (or paste S3 URL)
3. Preview the **resource graph** before creating
4. Enter parameter values
5. Acknowledge IAM resource creation (`CAPABILITY_IAM`)
6. Create stack — watch resources provision in real time

### Via CLI

```bash
# Create a new stack
aws cloudformation create-stack \
  --stack-name my-app-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=prod \
               ParameterKey=KeyName,ParameterValue=my-key \
  --capabilities CAPABILITY_NAMED_IAM

# Wait for completion
aws cloudformation wait stack-create-complete --stack-name my-app-stack

# Update an existing stack
aws cloudformation update-stack \
  --stack-name my-app-stack \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# Preview changes before updating (Change Set)
aws cloudformation create-change-set \
  --stack-name my-app-stack \
  --change-set-name my-changes \
  --template-body file://template.yaml

aws cloudformation describe-change-set \
  --stack-name my-app-stack \
  --change-set-name my-changes

aws cloudformation execute-change-set \
  --stack-name my-app-stack \
  --change-set-name my-changes

# Check stack status
aws cloudformation describe-stacks --stack-name my-app-stack
aws cloudformation describe-stack-events --stack-name my-app-stack

# Delete a stack (removes ALL resources it created)
aws cloudformation delete-stack --stack-name my-app-stack
```

---

## Stack Status Values

| Status | Meaning |
|--------|---------|
| `CREATE_IN_PROGRESS` | Resources are being created |
| `CREATE_COMPLETE` | All resources created successfully |
| `CREATE_FAILED` | Creation failed — check events for errors |
| `ROLLBACK_COMPLETE` | Failed and automatically rolled back |
| `UPDATE_IN_PROGRESS` | Update in progress |
| `UPDATE_COMPLETE` | Update successful |
| `UPDATE_ROLLBACK_COMPLETE` | Update failed and rolled back |
| `DELETE_IN_PROGRESS` | Resources are being deleted |
| `DELETE_COMPLETE` | All resources deleted |

---

## Drift Detection

Detect if resources were manually modified outside CloudFormation:

```bash
aws cloudformation detect-stack-drift --stack-name my-app-stack

# Check drift status
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <id-from-above>

# See which resources drifted
aws cloudformation describe-stack-resource-drifts \
  --stack-name my-app-stack \
  --stack-resource-drift-status-filters MODIFIED DELETED
```

---

## CloudFormation vs Terraform

| Feature | CloudFormation | Terraform |
|---------|---------------|-----------|
| Cloud support | AWS only | Multi-cloud |
| Language | YAML / JSON | HCL (HashiCorp Config Language) |
| State management | AWS-managed | State file (local or remote) |
| AWS integration | Native, deep | Via provider |
| Rollback | Automatic | Manual |
| Drift detection | Built-in | `terraform plan` |
| Ecosystem | AWS CDK (programmatic) | Extensive providers |
| Best for | AWS-only teams, deep AWS integration | Multi-cloud or existing Terraform users |

---

## AWS CDK (Alternative to Raw YAML)

The AWS Cloud Development Kit (CDK) lets you write CloudFormation infrastructure using **TypeScript, Python, Java, or Go** — then synthesizes it to a CloudFormation template:

```typescript
// CDK example (TypeScript)
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class MyStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);

    new s3.Bucket(this, 'MyBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });
  }
}
```

```bash
cdk synth   # generate CloudFormation YAML
cdk deploy  # deploy the stack
cdk diff    # see what will change
cdk destroy # delete the stack
```

CDK is recommended for complex infrastructure — you get IDE autocomplete, type safety, loops, conditionals, and reusable constructs.