---
title: Administer AWS Gateway cloud
linkTitle: Administer AWS Gateway cloud
draft: false
weight: 90
description: As a Cloud Administrator / Operator, you are responsible for
  configuring and managing your organization’s AWS infrastructure. This topic
  contains setup and test details for the additional AWS services that are
  required for Axway’s agents to govern your AWS API Gateway service. The
  additional services which will be configured are AWS CloudWatch, AWS SQS, AWS
  Config, and AWS Lambda.
---
## Overview

Connecting AWS API Gateway to AMPLIFY Central will provide you with a connected/managed environment, and a global centralized view of your APIs and their related traffic, allowing users to have a centralized governance (creation/deployment/publish/subscription) and monitoring of the traffic for AWS API Gateway hosted APIs.

Each AWS Gateway is represented by an AMPLIFY Central environment allowing you to better filter APIs and their traffic. Supplied with the environment, two agents, Discovery and Traceability, interact with AWS API Gateway and AMPLIFY Central.

### Discovery Agent

The Discovery Agent has two operating modes, continuous discovery and synchronous discovery. It is important to know which mode will be used prior to deploying the CloudFormation stack. The same procedure is used in both modes, however inputs, outputs, and services used will differ slightly.

#### Continuous Discovery Overview

To deploy an API In the AWS API Gateway, you create an API deployment and associate it with a stage. The Axway Discovery Agent listens for new deployments and for stage updates to existing deployments. When the agent receives an event it will publish, or update AMPLIFY Central with the API details. It is possible for the agent to publish the API information directly into the Unified Catalog or to be added to the environment associated with the agent in AMPLIFY Central.

In order for the Discovery Agent to receive the API details, the following AWS services are used:

| AWS Service    | Purpose                                                                                                                                                                                                                                                                                                                                   |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AWS Config     | Set up to monitor any configuration changes on API Gateway resources, specifically REST APIs and stages. When those changes are detected, they are sent to CloudWatch logs, and then they are sent to SQS.                                                                                                                                |
| AWS SQS        | The queue receives messages available to the Discovery Agent to find and determine what kind of resource that message is, what type of changes were made (update, delete, create), and it will query against API Gateway to get additional information about those changes, if needed, finally that info is sent to the AMPLIFY Platform. |
| AWS CloudWatch | Monitors resources and changes that the Discovery Agent made to the logging.                                                                                                                                                                                                                                                              |

![Service Discovery](/Images/central/connect-aws-gateway/aws-discovery-agent_v2.png)

The AWS Discovery Agent discovers newly created, previously undiscovered REST APIs, as well as changes to the APIs stage(s), which then updates the logging that enables the Traceability Agent (see below).

The agent only publishes APIs that pass the tagging criteria that is configured in the agent configuration file, see [Discover APIs](/docs/central/connect-aws-gateway/filtering-apis-to-be-discovered-1/). The agent will use the tags which are associated with the stage that is associated with the API.

As soon as an API is published to AMPLIFY Central, a new tag (APIC_ID) is added to the stage so that the Discovery Agent will not publish it again. The value of the APIC_ID tag is the ID of the resource representing the API in Central. It only discovers published APIs where the stage has one or more tags as defined in the agent configuration file.

#### Synchronous Discovery Overview

To deploy an API in the AWS API Gateway, you create an API deployment and associate it with a stage. The Axway Discovery Agent, when executed, will find all APIs and stages in AWS API Gateway and send them to AMPLIFY Central. Once synchronizing all resources, the agent will stop and no changes will be sent to AMPLIFY Central until it is started again.

This operating mode does not utilize the AWS Config, SQS, or CloudWatch services as the continuous mode does.

### Traceability Agent

The Traceability Agent sends summaries to AMPLIFY Central of the API traffic that has passed through the AWS API Gateway. The agent only sends a traffic summary for APIs that have been discovered (i.e. tagged with APIC_ID).

The Traceability Agent is used to filter the AWS CloudWatch logs that are related to discovered APIs and prepare the transaction events that are sent to AMPLIFY platform. Each time an API is called by a consumer it will result in an event (summary + detail) being sent to AMPLIFY Central. API Observer provides a view of the traffic and API usage of APIs deployed to the Gateway.

In order for the Traceability Agent to monitor API traffic, the following AWS services are used:

| AWS Service    | Purpose                                                                                                                                                                                                                                                                                                  |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AWS Lambda     | Runs code in response to events and automatically manages the computing resources required by that code. CloudWatch will write whenever a usage of an API invoked, and that is sent to the Lambda function, which parses out some pertinent information in order to track that usage and send it to SQS. |
| AWS SQS        | SQS messages are read by the Traceability Agent. The REST API ID and the stage ID are then queried back to CloudWatch for the additional transaction details (i.e. headers), in order to fully create a transaction object, which is then sent to the AMPLIFY platform.                                  |
| AWS CloudWatch | Monitors when an API is consumed, and if the Discovery Agent made changes to the logging. Those events are logged to CloudWatch.                                                                                                                                                                         |

The Traceability Agent requires read write access to SQS and read only access to CloudWatch.

![Service Discovery](/Images/central/connect-aws-gateway/aws-traceability-agent_v2.png)

The AWS service usage cost for the agents is explain below.

### Minimum requirements

* [AMPLIFY Central Service Account](/docs/central/connect-aws-gateway/prepare-amplify-central-1/#create-a-service-account)
* [API Key credentials on AWS](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html). Allowing for CLI access.
* [Amazon CloudWatch Service](https://aws.amazon.com/cloudwatch/)
* [Amazon Simple Queue Service](https://aws.amazon.com/sqs/)
* [AWS Lambda](https://aws.amazon.com/lambda/)
* Agent Config Package [downloaded](https://axway.bintray.com/generic-repo/aws-agents/aws_apigw_agent_config/latest/)

### CloudFormation templates

The agent config package contains a Lambda function (`traceability_lambda.zip`), and the CloudFormation templates.

Continuous discovery mode (`amplify-agents-deploy-all.yaml amplify-agents-resources.yaml amplify-agents-ec2.yaml`) which configure the additional AWS services (`CloudWatch, SQS, Lambda, and EC2`) that the agents require to function normally.

Synchronous discovery mode (`amplify-agents-deploy-all.yaml amplify-agents-resources.yaml`) which configure the additional AWS services (`AWS CloudWatch, AWS SQS, and AWS Lambda`) that the agents require to function normally.

Upload all of these resources to an S3 bucket, within the target region. Take note of the bucket name and URL to the `amplify-agents-deploy-all.yaml`.

If deploying the EC2 instance within these templates, additionally create the following file structure that the instance will retrieve for the agents.

```
amplify-agents-deploy-all.yaml
amplify-agents-resources.yaml
amplify-agents-ec2.yaml
resources
|    da_env_vars
|    ta_env_vars
|    keys
|    |    private_key.pem
|    |    public_key.pem
```

For the values in these **\*\_env_var** files see [Deploy your agents](/docs/central/connect-aws-gateway/deploy-your-agents-1)

For the key files see [Prepare AMPLIFY Central](/docs/central/connect-aws-gateway/prepare-amplify-central-1/)

As a Cloud Administrator, you may choose to manually create the IAM resources (jump to [CloudFormation (without IAM)](#cloudformation-without-iam)) or allow the CloudFormation the ability to do so. The resources created within IAM are three Roles, a Group, and a User.

For the list of minimum access rights for these CloudFormation templates see [Minimum rights for CloudFormation deployment](#minimum-rights-for-cloudformation-deployment)

#### Parameters (IAM and Resources)

The inputs to the IAM Setup CloudFormation Template (`amplify-agents-deploy-all.yaml`):

| Parameter Name                | Description                                                                                                                               | Default Value                   | Mode       |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- | ---------- |
| APIGWCWRoleSetup              | If set to true, the IAM Role for the API Gateway Service to write logs to CloudWatch will be created                                       | true                            | both       |
| ConfigServiceSetup            | If set to true, the IAM Role for the Config Service will be created                                                                        | true                            | continuous |
| ConfigBucketName              | The name of the bucket the Config Service, if enabled, will store AWS Config Logs. The account number and region will be appended to this | apigw-config-discovery          | continuous |
| ConfigBucketExists            | If set to true, the Config Bucket will not be created                                                                                      | false                           | continuous |
| DiscoveryQueueName            | The name of the Queue that the Discovery Agent will read from                                                                             | aws-apigw-discovery             | continuous |
| DiscoveryAgentLogGroupName    | The name that the Discovery Agent running on EC2 will log to                                                                              | amplify-discovery-agent-logs    | continuous |
| TraceabilityQueueName         | The name of the Queue that the Traceability Lambda will be writing to and the Traceability Agent will read from                           | aws-apigw-traceability          | both       |
| TraceabilityAgentLogGroupName | The name that the Traceability Agent running on EC2 will log to                                                                           | amplify-traceability-agent-logs | continuous |
| TraceabilityLogGroupName      | The name of the Traceability Log Group to be created                                                                                      | aws-apigw-traffic-logs          | both       |
| AgentResourcesBucket          | The S3 bucket that has the resources needed for deploying this stack, lambda and nested stack templates                                   |                                 | both       |
| EC2AgentDeploy                | If set to true, the Instance Role and Profile will be created to give to the EC2 Instance. If false, a User will be created                    | true                            | continuous |
| EC2StackVPCID                 | The VPC to deploy the EC2 instance to. Leave blank to deploy all infrastructure                                                           |                                 | continuous |
| EC2StackSecurityGroup         | The Security Group to assign to the EC2 instance. Not needed when deploying all infrastructure                                            |                                 | continuous |
| EC2StackSubnet                | The Subnet the EC2 instance will be in. Not needed when deploying all infrastructure                                                      |                                 | continuous |
| EC2StackKeyName               | The SSH Key to deploy inside the EC2 instance                                                                                             |                                 | continuous |
| EC2StackInstanceType          | The instance type to use for this EC2 instance                                                                                            | t1.micro                        | continuous |
| EC2StackSSHLocation           | The CIDR range that is allowed to SSH to the instance                                                                                     | 0.0.0.0/0                       | continuous |
| EC2StackPublicIPAddress       | Assign a Public IP address, the agents needs internet access for AMPLIFY Central communication                                            | true                            | continuous |

#### Resources (IAM and Resources)

The resources created by the CloudFormation template:

| Resource Type                         | Resource Name                     | Condition                                                | Description                                                                                        | Operating Mode |
| ------------------------------------- | --------------------------------- | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | -------------- |
| AWS::IAM::Role                        | ConfigServiceIAMRole              | ConfigServiceSetup = true                                | Creates the IAM Role for the Config Service                                                        | continuous     |
| AWS::IAM::Role                        | TraceabilityLambdaIAMRole         |                                                          | Creates the IAM Role for the Traceability Lambda Function                                          | both           |
| AWS::IAM::Role                        | TraceabilityAPIGWCWIAMRole        | APIGWCWRoleSetup = true                                  | Creates the IAM Role for API Gateway to write logs to CloudWatch                                   | both           |
| AWS::IAM::ManagedPolicy               | APICAgentsPolicy                  |                                                          | Creates the IAM Policy that the agents will utilize                                                | both           |
| AWS::IAM::Role                        | AgentsInstanceRole                | EC2AgentDeploy = true                                    | Creates the IAM Instance Role to assign to the IAM instance role                                   | continuous     |
| AWS::IAM::InstanceProfile             | AgentsInstanceProfile             | EC2AgentDeploy = true                                    | Creates the IAM Instance Profile to assign to the EC2 instance                                     | continuous     |
| AWS::IAM::APICAgentsUser              | APICAgentsUser                    | EC2AgentDeploy = false                                   | Creates the IAM User to generate keys for to supply to the agents                                  | both           |
| AWS::CloudFormation::Stack            | ResourcesStack                    |                                                          | Creates the Resources Stack for the APIC Agents                                                    | both           |
| AWS::CloudFormation::Stack            | EC2Stack                          | EC2AgentDeploy = true                                    | Creates the EC2 Stack for the APIC Agents                                                          | continuous     |
| **Nested Stack Resources**          |                                   |                                                          |                                                                                                    |                |
| amplify-agents-resources.yaml         |                                   |                                                          |                                                                                                    | both           |
| AWS::Config::ConfigurationRecorder    | DiscoveryConfigRecorder           | ConfigServiceSetup = true                                | The setup needed to have Config start recording changes                                            | continuous     |
| AWS::Config::DeliveryChannel          | DiscoveryConfigDeliveryChannel    | ConfigServiceSetup = true                                | The delivery channel used by Config to send the configurations to ConfigBucket                     | continuous     |
| AWS::S3::Bucket                       | DiscoveryConfigBucket             | ConfigServiceSetup = true and ConfigBucketExists = false | The S3 bucket used to store all current configurations, required for Config                        | continuous     |
| AWS::Events::Rule                     | DiscoveryConfigCloudWatchRule     |                                                          | The rule used to only send API Gateway Config changes to the SqsQueue                              | continuous     |
| AWS::SQS::Queue                       | DiscoveryConfigSqsQueue           |                                                          | The Queue that all API Gateway configuration changes are pushed to                                 | continuous     |
| AWS::SQS::QueuePolicy                 | DiscoveryConfigSqsQueuePolicy     |                                                          | The policy that grants permission to push to the SqsQueue                                          | continuous     |
| AWS::Logs::LogGroup                   | TraceabilityAccessLogGroup        |                                                          | Creates the Log Group to track access of APIC tracked API Gateway endpoints                        | both           |
| AWS::ApiGateway::Account              | TraceabilityAPIGWCWRole           | TraceabilityAPIGWCWRoleArn not blank                     | Sets the CloudWatch Role ARN in the API Gateway Settings                                           | both           |
| AWS::Lambda::Function                 | TraceabilityLambda                |                                                          | The Lambda function that takes Cloud Watch events in the Log Group and sends them to the SQS Queue | both           |
| AWS::Lambda::Permission               | TraceabilityLambdaCWInvoke        |                                                          | Allows Cloud Watch events to trigger the Lambda Function                                           | both           |
| AWS::SQS::Queue                       | TraceabilitySqsQueue              |                                                          | The Queue that all API Gateway access logs are pushed to                                           | both           |
| AWS::Logs::SubscriptionFilter         | TraceabilityLogToLambdaFilter     |                                                          | Filter events from the Tracebaility Logs to the Lambda Function                                    | both           |
| amplify-agents-ec2.yaml               |                                   |                                                          |                                                                                                    | continuous     |
| AWS::EC2::VPC                         | AgentsVPC                         | VpcID not blank                                          | The VPC the instance will be deployed to                                                           | continuous     |
| AWS::EC2::Subnet                      | AgentsSubnet                      | VpcID not blank                                          | The Subnet to deploy the EC2 instance in                                                           | continuous     |
| AWS::EC2::RouteTable                  | AgentsRouteTable                  | VpcID not blank                                          | The route table for defining routes for the VPC                                                    | continuous     |
| AWS::EC2::SubnetRouteTableAssociation | AgentsRouteTableSubnetAssociation | VpcID not blank                                          | Associates the Subnet with the VPC and Route Table                                                 | continuous     |
| AWS::EC2::Route                       | AllowInternetTraffic              | VpcID not blank                                          | The route to direct all traffic in the VPC to the internet                                         | continuous     |
| AWS::EC2::SecurityGroup               | AgentsSecurityGroup               | VpcID not blank                                          | The Security Group assigned to the VPC, allowing SSH only to the host                               | continuous     |
| AWS::EC2::InternetGateway             | InternetGW                        | VpcID not blank                                          | The Internet Gateway so the agents can talk to AMPLIFY Central                                     | continuous     |
| AWS::EC2::VPCGatewayAttachment        | GatewayAttachement                | VpcID not blank                                          | Attaches the Internet Gateway to the VPC                                                           | continuous     |
| AWS::EC2::Instance                    | AgentsHost                        |                                                          | The EC2 instance that the agents wil run in                                                        | continuous     |

#### Outputs (IAM and Resources)

The outputs from the CloudFormation template:

| Output Name           | Description                                                   | Operating Mode |
| --------------------- | ------------------------------------------------------------- | -------------- |
| DiscoveryQueueName    | Amazon SQS Queue name containing the changes to API Gateway.  | continuous     |
| TraceabilityQueueName | Amazon SQS Queue name containing the API Gateway Access Logs. | both           |

These outputs will used as inputs for running both the Discovery and Traceability agents.

#### Access Credentials (optional)

If not deploying within an EC2 instance, an Access and Secret Key for the user should be created and given to the person deploying the agent. Please follow your organizations key rotation policies. See [Managing Access Keys for IAM Users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html).

### CloudFormation (without IAM)

Before deploying the resources template, the following IAM resources should be created and ARN given to the user deploying the CloudFormation template.

For the list of minimum access rights for these CloudFormation templates, see [Minimum rights for CloudFormation deployment](#minimum-rights-for-cloudformation-deployment).

#### Required IAM Resources

If you prefer to create IAM Resources outside of the CloudFormation script, the following must be created:

##### ConfigServiceIAMRole (Continuous Discovery mode)

This role is used by the Config Service to monitor configuration changes to AWS API gateway Resources. It must have the following policies.

Assume Role Policies

| Service              | Description                                                   |
| -------------------- | ------------------------------------------------------------- |
| config.amazonaws.com | This assume role policy lets the config service use this role |

Managed Policies

| Managed Policy ARN                                 | Description                                                                                                                    |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| arn:aws:iam::aws:policy/service-role/AWSConfigRole | This policy is needed for the Config Service to be able to read the various configurations to track for changes in API Gateway |

Policies

| Effect | Action          | Resource          | Description                                                                    |
| ------ | --------------- | ----------------- | ------------------------------------------------------------------------------ |
| Allow  | s3:GetBucketAcl | Config Bucket ARN | This allows the Config Service to get the access control list of the S3 bucket |
| Allow  | s3:PutObject    | Config Bucket ARN | This allows the Config Service to save configurations to the S3 bucket         |
| Allow  | config:Put\*    | \*                | This allows access to all config put actions for tracking changes              |

##### TraceabilityLambdaIAMRole

This role is used by the Traceability Lambda so it may be triggered by CloudWatch and then write its output to the Traceability SQS queue

Assume Role Policies

| Service              | Description                                                                                 |
| -------------------- | ------------------------------------------------------------------------------------------- |
| lambda.amazonaws.com | This assume role policy lets the lambda service, for the Traceability Lambda, use this role |

Managed Policies

| Managed Policy ARN                                               | Description                                                                            |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole | This policy is needed so the Traceability Lambda can execute and write CloudWatch logs |

Policies

| Effect | Action          | Resource               | Description                                                                         |
| ------ | --------------- | ---------------------- | ----------------------------------------------------------------------------------- |
| Allow  | sqs:GetQueueUrl | Traceability Queue ARN | This policy is needed so the Traceability Lambda can connect to an SQS Queue        |
| Allow  | sqs:SendMessage | Traceability Queue ARN | This policy is needed so the Traceability Lambda can write messages to an SQS Queue |

##### TraceabilityAPIGWCWIAMRole

This role is used by AWS APIGW to save its usage events to an AWS CloudWatch log.

Assume Role Policies

| Service                  | Description                                                        |
| ------------------------ | ------------------------------------------------------------------ |
| apigateway.amazonaws.com | This assume role policy lets the API Gateway service use this role |

Managed Policies

| Managed Policy ARN                                                        | Description                                                                   |
| ------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs | This policy is needed so the API Gateway service can write to CloudWatch logs |

##### APICAgentsPolicy

The group that has all of the policies attached which are needed by the APICAgentsUser

Policies

| Effect | Action                  | Resource                                                           | Description                                                                                                     |
| ------ | ----------------------- | ------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| Allow  | logs:DescribeLogGroups  | All log groups in AWS account and Region                           | Used to validate the connection to AWS CloudWatch on Agent Startup                                                  |
| Allow  | logs:DescribeLogStreams | API-Gateway-Execution-Logs\_\* log groups in account and region    | Allows the agent to get the streams for API Gateway Execution Logs                                               |
| Allow  | logs:GetLogEvents       | API-Gateway-Execution-Logs\_\* log groups in account and region    | Allows the agent to get log events for API Gateway Execution Logs                                               |
| Allow  | logs:FilterLogEvents    | API-Gateway-Execution-Logs\_\* log groups in account and region    | Allows the agent to get log events for specific transactions in API Gateway                                     |
| Allow  | logs:PutLogEvents       | Log streams for the discovery and traceability agents log group    | Allows the agent, via docker on the EC2 instance, to log to CloudWatch                                               |
| Allow  | logs:CreateLogGroup     | Log streams for the discovery and traceability agents log group    | Allows the agent, via docker on the EC2 instance, to create its log group in CloudWatch                              |
| Allow  | logs:CreateLogStream    | Log streams for the discovery and traceability agents log group    | Allows the agent, via docker on the EC2 instance, to create a log stream in CloudWatch                               |
| Allow  | logs:DescribeLogStreams | The discovery and traceability agents log group                    | Allows the agent, via docker on the EC2 instance, to determine if it needs to create a log group on CloudWatch       |
| Allow  | sqs:DeleteMessage       | Traceability and Discovery Queue ARNs                              | Allows the agent to read messages from the SQS Queues                                                           |
| Allow  | sqs:GetQueueUrl         | Traceability and Discovery Queue ARNs                              | Allows the agent to get the URL of the Queue in order to read messages                                          |
| Allow  | sqs:ReceiveMessage      | Traceability and Discovery Queue ARNs                              | Allows the agent to remove messages after processing them                                                       |
| Allow  | apigateway:PUT          | All API Gateway Resources in region                                | Allows the Discovery Agent to make updates to the API Endpoints                                                 |
| Allow  | apigateway:PATCH        | All API Gateway Resources in region                                | Allows the Discovery Agent to make updates to the API Endpoints                                                 |
| Allow  | apigateway:GET          | All API Gateway Resources in region                                | Allows the Discovery Agent to get the configuration of the API Endpoints                                        |
| Allow  | apigateway:DELETE       | All API Gateway Resources in region                                | Allows the Discovery Agent to remove tags for Unsubscribe events                                                   |
| Allow  | s3:ListBucket           | The bucket set as the AgentsResourceBucket when creating the stack | Used by the EC2 instance to list all files in the resources directory of the bucket, for the agent execution    |
| Allow  | s3:GetObjectAcl         | The bucket set as the AgentsResourceBucket when creating the stack | Used by the EC2 instance to get all the files in the resources directory of the bucket, for the agent execution |
| Allow  | s3:GetObject            | The bucket set as the AgentsResourceBucket when creating the stack | Used by the EC2 instance to get all the files in the resources directory of the bucket, for the agent execution |

##### AgentsInstanceRole

The role that is assigned to the EC2 instance profile so the agents may access the APICAgentsPolicy

Assume Role Policies

| Service           | Description                                                                                                    |
| ----------------- | -------------------------------------------------------------------------------------------------------------- |
| ec2.amazonaws.com | Allows the EC2 instance that will run the agents the ability to access the services the agents need to execute |

Managed Policy

| Managed Policy     | Description              |
| ------------------ | ------------------------ |
| AgentsInstanceRole | The role described above |

##### AgentsInstanceProfile

The instance profile that will be assigned to the EC2 instance

Role

| Role               | Description              |
| ------------------ | ------------------------ |
| AgentsInstanceRole | The role described above |

##### APICAgentsUser

The user that the APIC Agents will log in as, added to the APICAgentsGroup

An Access and Secret Key for the user should be created and given to the person deploying the agent, please follow your organizations key rotation policies. See [Managing Access Keys for IAM Users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)

#### Parameters (Resources only)

The inputs to the Resource CloudFormation template (`amplify-agents-resources.yaml`):

| Parameter Name             | Description                                                                                                                                | Default Value           | Operating Mode |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------- | -------------- |
| SetupConfigService         | This parameter is used to disable the configuration of AWS Config Service, and all of its dependencies, while building the stack.          | true                    | continuous     |
| ConfigBucketName           | The name of the bucket the Config Service, if enabled, will store AWS Config Logs. The account number and region will be appended to this. | apigw-config-discovery  | continuous     |
| ConfigBucketExists         | If set to true, the Config Bucket will not be created.                                                                                      | false                   | continuous     |
| ConfigServiceRoleArn       | The ARN for the Config Service IAM Role.                                                                                                   |                         | continuous     |
| DiscoveryQueueName         | The name of the queue that will hold only changes made to API Gateway resources. The region will be appended to this.                      | aws-apigw-discovery     | continuous     |
| TraceabilityAPIGWCWRoleArn | The ARN for the IAM role that allows API Gateway the permission to write CloudWatch logs. Leave blank if this does not need to be configured.    |                         | both           |
| TraceabilityLambdaRoleArn  | The Log Group created to track access of APIC tracked API Gateway endpoints.                                                               |                         | both           |
| TraceabilityLogGroupName   | The CloudWatch Log Group that will store transaction details for AWS API Gateway usage events.                                              | APIGW_Traceability_Logs | both           |
| TraceabilityFunctionBucket | The S3 bucket that has the executable for the traceability lambda function.                                                                |                         | both           |
| TraceabilityFunctionKey    | The key of the traceability lambda function in the bucket.                                                                                 |                         | both           |
| TraceabilityQueueName      | The name of the queue that will hold traceability logs of API Gateway resources. The region will be appended to this.                      | aws-apigw-traceability  | both           |

#### Resources (Resources only)

The services that are configured with this CloudFormation template:

| Resource Type                      | Resource Name                  | Condition                 | Description                                                                                         | Operating Mode |
| ---------------------------------- | ------------------------------ | ------------------------- | --------------------------------------------------------------------------------------------------- | -------------- |
| AWS::Config::ConfigurationRecorder | DiscoveryConfigRecorder        | ShouldSetupConfigService  | The setup needed to have Config start recording changes.                                            | continuous     |
| AWS::Config::DeliveryChannel       | DiscoveryConfigDeliveryChannel | ShouldSetupConfigService  | The delivery channel used by Config to send the configurations to ConfigBucket.                     | continuous     |
| AWS::S3::Bucket                    | DiscoveryConfigBucket          | ShouldCreateConfigBucket  | The S3 bucket used to store all current configurations, required for Config.                        | continuous     |
| AWS::Events::Rule                  | DiscoveryConfigCloudWatchRule  |                           | DiscoveryConfigCloudWatchRule.                                                                      | continuous     |
| AWS::SQS::Queue                    | DiscoveryConfigSqsQueue        |                           | The Queue that all API Gateway configuration changes are pushed to.                                 | continuous     |
| AWS::SQS::QueuePolicy              | DiscoveryConfigSqsQueuePolicy  |                           | The policy that grants permission to push to the SqsQueue.                                          | continuous     |
| AWS::Logs::LogGroup                | TraceabilityAccessLogGroup     |                           | Creates the Log Group to track access of APIC tracked API Gateway endpoints.                        | both           |
| AWS::ApiGateway::Account           | TraceabilityAPIGWCWRole        | ShouldSetupAPIGWCWRoleArn | Sets the CloudWatch Role ARN in the API Gateway Settings.                                           | both           |
| AWS::Lambda::Function              | TraceabilityLambda             |                           | The Lambda function that takes Cloud Watch events in the Log Group and sends them to the SQS Queue. | both           |
| AWS::Lambda::Permission            | TraceabilityLambdaCWInvoke     |                           | Allows Cloud Watch events to trigger the Lambda Function.                                           | both           |
| AWS::SQS::Queue                    | TraceabilitySqsQueue           |                           | The Queue that all API Gateway access logs are pushed to.                                           | both           |
| AWS::Logs::SubscriptionFilter      | TraceabilityLogToLambdaFilter  |                           | Filter events from the Traceability Logs to the Lambda Function.                                    | both           |

#### Outputs (Resources only)

The outputs from the Resource CloudFormation template:

| Output Name           | Description                                                   | Operating Mode |
| --------------------- | ------------------------------------------------------------- | -------------- |
| DiscoveryQueueName    | Amazon SQS Queue name containing the changes to API Gateway.  | continuous     |
| TraceabilityQueueName | Amazon SQS Queue name containing the API Gateway Access Logs. | both           |

These outputs will be used as inputs for running both the Discovery and Traceability agents

### Testing the setup for the AWS Discovery Agent

* Make a change to an existing REST API/Stage OR create a new REST API/Stage
* Validate that the `<DiscoveryQueueName>` has messages waiting

### Testing the setup for the AWS Traceability Agent

{{< alert title="Note" color="primary" >}}To test the AWS Traceability Agent setup, the AWS Discovery Agent should be running and have discovered an API, this is required as the Stage config is updated to log transactional data.{{< /alert >}}

* Send traffic through an API that the DA has discovered
* Validate messages received in AWS SQS
* Validate logging in CloudWatch under the Log group named after the REST API ID/Stage

### Connecting AWS API Gateway to AMPLIFY Central QuickStart

* [Prepare AWS API Gateway](/docs/central/connect-aws-gateway/prepare-aws-api-gateway/)
* [Prepare AMPLIFY Central](/docs/central/connect-aws-gateway/prepare-amplify-central-1/)
* [Deploy agents](/docs/central/connect-aws-gateway/deploy-your-agents-1/)

### Agents AWS Cost

The AWS cost of the agent will depend on your traffic usage. The more traffic you have the more it will cost.
You can follow your cost using the AWS [Cost management service](https://console.aws.amazon.com/cost-management/home#/dashboard).
The following are the costs of the AWS services the Agents rely on:

* AWS [Cloud formation pricing](https://aws.amazon.com/cloudformation/pricing/)</br>
  The two CloudFormation templates do not add additional charges, as they are used only once and create only AWS:: resources.
* AWS [S3 bucket pricing](https://aws.amazon.com/s3/pricing/)</br>
  One bucket is needed at installation time for storing a lambda. The file size is less than 4Mo. This bucket is accessed only once at the installation time.
  It has negligible cost.
* AWS [Config pricing](https://aws.amazon.com/config/pricing/)</br>
  You pay \$0.003 per configuration item recorded in your AWS account per AWS Region. This is dependant on the number of changes in API / stage you will perform in a month. We don't set up rules or conformance pack. Here is the list of resources the agent needs to monitor: _ApiGateway:RestAPI_, _ApiGateway:Stage_, _ApiGatewayV2:RestAPI_ and _ApiGateway:Stage_.
* AWS [Lambda pricing](https://aws.amazon.com/lambda/pricing/)</br>
  You are charged based on the number of requests for your functions and the duration (the time it takes for your code to execute). The AWS Lambda free usage tier includes 1M free requests per month and 400,000 GB-seconds of compute time per month. Our traceability lambda is called each time one of the discovered APIs is consumed. The amount of allocated memory for the lambda is set to 128Mo. The lambda runs on average within .5 second.</br>
  Lambda cost is based on following formulas:

    * Monthly cost charge: (# lambda call \* lambda execution time \* (lambda memory / 1024) - 400,000freeGB-s) \* **0.0000166667**
    * Monthly request charge: ((# lambda call - 1M free request) \* **0.0000002**)</br>
    * Samples:</br>
        * 2 million calls: monthly cost ($0) + monthly request charge ($0.20) = \$0.20</br>
        * 10 million calls: monthly cost ($10.42) + monthly request charge ($1.80) = \$12.22</br>

* AWS [CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)</br>
  You should be able to operate with the free tier, as the agent requires only one monitoring metrics (APIGW_Traceability_Logs).
* AWS [Simple Queue Service pricing](https://aws.amazon.com/sqs/pricing/)</br>
  Two standard queues are set up: one for Discovery Agent and one for Traceability Agent. The Discovery Agent queue will contain every stage deployment. The Traceability Agent queue will contain every call to discovered APIs. One million Amazon SQS requests for free each month. After free tier, it cost \$0.40 per million requests.
* AWS [API Gateway pricing](https://aws.amazon.com/api-gateway/pricing/)</br>
  The Amazon API Gateway free tier includes one million API calls received for REST APIs, one million API calls received for HTTP APIs, and one million messages and 750,000 connection minutes for WebSocket APIs per month for up to 12 months. If you exceed this number of calls per month, you will be charged the API Gateway usage rates. There are different rates based on the API type (HTTP / REST / Websocket).

Summary:

| AWS Service          | Cost in USD per month                                                                                                                                             |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cloud Formation      | 0                                                                                                                                                                 |
| S3 bucket            | 0                                                                                                                                                                 |
| Config               | 0.003 \* (# config)                                                                                                                                               |
| Lambda execution     | ((# lambda call \* lambda execution time \* (lambda memory / 1024) - 400,000freeGB-s) \* **0.0000166667**) + ((# lambda call - 1M free request) \* **0.0000002**) |
| CloudWatch           | 0                                                                                                                                                                 |
| Simple Queue Service | One million requests free or \$0.40 per million requests thereafter                                                                                               |
| API Gateway          | refer to [API Gateway pricing](https://aws.amazon.com/api-gateway/pricing/) for details                                                                           |

### Minimum rights for CloudFormation deployment

The required privileges to deploy the CloudFormation templates to the AWS platform are listed below.

Note that these privileges do not include those necessary for rollback, if the stack fails, or removing the stack if needed.

| Action                                | Resources                                                                                       | Mode       | Template                  | Condition                 |
| ------------------------------------- | ----------------------------------------------------------------------------------------------- | ---------- | ------------------------- | ------------------------- |
| cloudformation:ListStackResources     | The stacks being created                                                                        | both       | all                       |                           |
| cloudformation:ListStacks             | The stacks being created                                                                        | both       | all                       |                           |
| cloudformation:DescribeStackResource  | The stacks being created                                                                        | both       | all                       |                           |
| cloudformation:DescribeStackResources | The stacks being created                                                                        | both       | all                       |                           |
| cloudformation:GetTemplateSummary     | The stacks being created                                                                        | both       | all                       |                           |
| cloudformation:CreateStack            | The stacks being created                                                                        | both       | all                       |                           |
| s3:ListBuckets                        | ConfigBucketName, AgentResourcesBucket                                                          | both       | all                       |                           |
| s3:GetObject                          | traceability_lambda.zip, CloudFormation templates                                               | both       | amplify-agents-resources  |                           |
| s3:CreateBucket                       | ConfigBucketName-<AWS::AccountID>-<AWS::Region>                                                 | continuous | amplify-agents-resources  | ConfigServiceSetup = true |
| apigateway:PATCH                      | /account                                                                                        | both       | amplify-agents-resources  | APIGWCWRoleSetup = true   |
| lambda:GetFunction                    | TraceabilityLambda                                                                              | both       | amplify-agents-resources  |                           |
| lambda:GetFunctionConfiguration       | TraceabilityLambda                                                                              | both       | amplify-agents-resources  |                           |
| lambda:CreateFunction                 | TraceabilityLambda                                                                              | both       | amplify-agents-resources  |                           |
| lambda:AddPermission                  | TraceabilityLambda                                                                              | both       | amplify-agents-resources  |                           |
| sqs:GetQueueAttributes                | DiscoveryConfigSqsQueue, TraceabilitySqsQueue                                                   | both       | amplify-agents-resources  |                           |
| sqs:CreateQueue                       | DiscoveryConfigSqsQueue, TraceabilitySqsQueue                                                   | both       | amplify-agents-resources  |                           |
| sqs:SetQueueAttributes                | DiscoveryConfigSqsQueue, TraceabilitySqsQueue                                                   | both       | amplify-agents-resources  |                           |
| logs:DescribeLogGroups                | DiscoveryAgentLogGroupName, TraceabilityAgentLogGroupName, TraceabilityLogGroupName             | both       | amplify-agents-deploy-all |                           |
| logs:CreateLogGroup                   | DiscoveryAgentLogGroupName, TraceabilityAgentLogGroupName, TraceabilityLogGroupName             | both       | amplify-agents-resources  |                           |
| logs:PutSubscriptionFilter            | TraceabilityLogGroupName                                                                        | both       | amplify-agents-resources  |                           |
| config:DescribeConfigurationRecorders | \*                                                                                              | continuous | amplify-agents-resources  | ConfigServiceSetup = true |
| config:DescribeDeliveryChannels       | \*                                                                                              | continuous | amplify-agents-resources  | ConfigServiceSetup = true |
| config:PutConfigurationRecorders      | \*                                                                                              | continuous | amplify-agents-resources  | ConfigServiceSetup = true |
| config:PutDeliveryChannel             | \*                                                                                              | continuous | amplify-agents-resources  | ConfigServiceSetup = true |
| config:StartConfigurationRecorder     | \*                                                                                              | continuous | amplify-agents-resources  | ConfigServiceSetup = true |
| events:DescribeRule                   | \*                                                                                              | continuous | amplify-agents-resources  | ConfigServiceSetup = true |
| events:PutRule                        | \*                                                                                              | continuous | amplify-agents-resources  | ConfigServiceSetup = true |
| events:PutTargets                     | \*                                                                                              | continuous | amplify-agents-resources  | ConfigServiceSetup = true |
| ec2:DescribeInstances                 | AgentsHost                                                                                      | continuous | amplify-agents-ec2        |                           |
| ec2:RunInstances                      | AgentsHost                                                                                      | continuous | amplify-agents-ec2        |                           |
| ec2:AssociateIamInstanceProfile       | AgentsHost                                                                                      | continuous | amplify-agents-ec2        |                           |
| ec2:DescribeKeyPairs                  | Key(s) that can be added to the Instance                                                        | continuous | amplify-agents-ec2        |                           |
| ec2:DecsribeVpcs                      | AgentsVPC or parameter VpcId (EC2StackVPCID)                                                    | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:CreateVpc                         | AgentsVPC                                                                                       | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:ModifyVpcAttribute                | AgentsVPC                                                                                       | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:AssociateRouteTable               | AgentsVPC                                                                                       | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:AttachInternetGateway             | AgentsVPC                                                                                       | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:CreateSecurityGroups              | AgentsSecurityGroup                                                                             | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:AuthorizeSecurityGroupIngress     | AgentsSecurityGroup                                                                             | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:DescribeSecurityGroups            | AgentsSecurityGroup or parameter SecurityGroup (EC2StackSecurityGroup)                          | continuous | amplify-agents-ec2        |                           |
| ec2:CreateSubnet                      | AgentsSubnet                                                                                    | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:DescribeSubnets                   | AgentsSubnet or parameter Subnet (EC2StackSubnet)                                               | continuous | amplify-agents-ec2        |                           |
| ec2:CreateInternetGateway             | AgentsInternetGW                                                                                | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:DecsribeInternetGateways          | AgentsInternetGW                                                                                | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:DescribeAvailabilityZones         | Availability Zones in Region                                                                    | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:DescribeRouteTables               | AgentsRouteTable                                                                                | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:CreateRouteTable                  | AgentsRouteTable                                                                                | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:CreateRoute                       | AgentAllowInternetTraffic                                                                       | continuous | amplify-agents-ec2        | VpcId = ""                |
| ec2:DescribeAccountAttributes         | AWS Account                                                                                     | continuous | amplify-agents-ec2        | VpcId = ""                |
| iam:ListRoles                         | ConfigServiceIAMRole, TraceabilityLambdaIAMRole, TraceabilityAPIGWCWIAMRole, AgentsInstanceRole | both       | amplify-agents-deploy-all |                           |
| iam:GetRole                           | ConfigServiceIAMRole, TraceabilityLambdaIAMRole, TraceabilityAPIGWCWIAMRole, AgentsInstanceRole | both       | amplify-agents-deploy-all |                           |
| iam:CreateRole                        | ConfigServiceIAMRole, TraceabilityLambdaIAMRole, TraceabilityAPIGWCWIAMRole, AgentsInstanceRole | both       | amplify-agents-deploy-all |                           |
| iam:PassRole                          | ConfigServiceIAMRole, TraceabilityLambdaIAMRole, TraceabilityAPIGWCWIAMRole, AgentsInstanceRole | both       | amplify-agents-deploy-all |                           |
| iam:GetRolePolicy                     | APICAgentsPolicy                                                                                | both       | amplify-agents-deploy-all |                           |
| iam:GetUserPolicy                     | APICAgentsPolicy                                                                                | both       | amplify-agents-deploy-all | EC2AgentDeploy = false    |
| iam:CreatePolicy                      | APICAgentsPolicy                                                                                | both       | amplify-agents-deploy-all |                           |
| iam:PutRolePolicy                     | APICAgentsPolicy                                                                                | both       | amplify-agents-deploy-all |                           |
| iam:GetUser                           | APICAgentsUser                                                                                  | both       | amplify-agents-deploy-all | EC2AgentDeploy = false    |
| iam:CreateUser                        | APICAgentsUser                                                                                  | both       | amplify-agents-deploy-all | EC2AgentDeploy = false    |
| iam:AttachUserPolicy                  | APICAgentsUser                                                                                  | both       | amplify-agents-deploy-all | EC2AgentDeploy = false    |
| iam:CreateInstanceProfile             | AgentsInstanceProfile                                                                           | continuous | amplify-agents-deploy-all | EC2AgentDeploy = true     |
| iam:AddRoleToInstanceProfile          | AgentsInstanceRole                                                                              | continuous | amplify-agents-deploy-all | EC2AgentDeploy = true     |
| iam:AttachRolePolicy                  | AgentsInstanceRole                                                                              | both       | amplify-agents-deploy-all | EC2AgentDeploy = true     |

### Troubleshooting

| Question                                        | Answer                                                                                                                                                                                              |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Why isn’t my API discovered?                    | Check that the tag set on the stage has a correct name and value based on the AWS_FILTER variable. See [Discover APIs](/docs/central/connect-aws-gateway/filtering-apis-to-be-discovered-1/).       |
| Why can’t my agents connect to AWS API Gateway? | Go to AWS console / IAM service and make sure that `AWS_REGION`, `AWS_AUTH_ACCESSKEY` and `AWS_AUTH_SECRETKEY` are valid and not inactivated.                                                       |
| Why can’t my agents connect to AMPLIFY Central? | Go to **AMPLIFY Central UI > Access > Service Accounts** and make sure that the Service Account is correctly named and valid. Make sure that the organizationID and team configuration are correct. |
| Why don’t I see traffic in AMPLIFY Central?     | Make sure that the Condor URL is accessible from the machine where Traceability Agent is installed.                                                                                                 |
| How to verify that the Agent is running?        | `docker inspect --format='{{json .State.Health}}' <container>`                                                                                                                                      |
