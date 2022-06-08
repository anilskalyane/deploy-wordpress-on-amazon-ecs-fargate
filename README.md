# Deploy-WordPress-on-amazon-ecs-fargate

Run WordPress without servers using Amazon ECS on AWS Fargate. We will create a VPC and an ECS cluster in which our WordPress tasks will run. An RDS MySQL instance will provide the database that WordPress requires. WordPress will persist its data on a shared Amazon EFS filesystem.

#### Prerequisites
You will need [AWS CLI version 2] to complete the tutorial. We have tested the steps on Amazon Linux 2.

Let’s start by setting a few environment variables:

```bash
export WOF_AWS_REGION=ap-south-1 <-- Change this to match your region
export WOF_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
export WOF_ECS_CLUSTER_NAME=ecs-fargate-wordpressexport WOF_CFN_STACK_NAME=WordPress-on-Fargate
```

#### Create base infrastructure
Next, we will create a VPC with two public and private subnets across two Availability Zones (AZ) and a NAT Gateway in each AZ. We have created a CloudFormation template that provisions the VPC resources, an RDS database, an EFS filesystem, an Application Load Balancer (ALB), a listener, target groups, and associated security groups.

Create a stack using CloudFormation:

```
aws cloudformation create-stack \
  --stack-name $WOF_CFN_STACK_NAME \
  --region $WOF_AWS_REGION \
  --template-body file://wordpress-ecs-fargate.yaml
```
The stack deployment takes roughly five minutes to complete.

The following commands waits until the stack deployment completes. You can use this command to determine when the stack deployment is complete or you can use the [AWS Management Console] to view deployment progress:

```
aws cloudformation wait stack-create-complete \
  --stack-name $(aws cloudformation describe-stacks  \
    --region $WOF_AWS_REGION \
    --stack-name $WOF_CFN_STACK_NAME \
    --query 'Stacks[0].StackId' --output text) \
  --region $WOF_AWS_REGION
```

Load environment variables from the CloudFormation stack’s output:

```
export WOF_EFS_FS_ID=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='EFSFSId'].OutputValue" \
  --output text)

export WOF_EFS_AP=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='EFSAccessPoint'].OutputValue" \
  --output text)
  
export WOF_RDS_ENDPOINT=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='RDSEndpointAddress'].OutputValue" \
  --output text)
  
 export WOF_VPC_ID=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='VPCId'].OutputValue" \
  --output text)
  
export WOF_PUBLIC_SUBNET0=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='PublicSubnet0'].OutputValue" \
  --output text)
  
export WOF_PUBLIC_SUBNET1=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='PublicSubnet1'].OutputValue" \
  --output text)
  
export WOF_PRIVATE_SUBNET0=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='PrivateSubnet0'].OutputValue" \
  --output text)

export WOF_PRIVATE_SUBNET1=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='PrivateSubnet1'].OutputValue" \
  --output text)
  
export WOF_ALB_SG_ID=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='ALBSecurityGroup'].OutputValue" \
  --output text)
  
export WOF_TG_ARN=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='WordPressTargetGroup'].OutputValue" \
  --output text)
Bash
```

The stack creates an EFS filesystem with mount targets in two AZs. WordPress tasks in each AZ will mount the EFS file system using the local EFS mount target in that AZ.

Note that the RDS MySQL instance this stack creates is a single point of failure. In production environments, we recommend improving the resilience of the WordPress database by using RDS Multi-AZ Deployments. You can also use Amazon Aurora Serverless instead of RDS MySQL, which automatically starts up, shuts down, and scales database capacity based on your application’s needs. It allows you to run your database without managing servers.

You can further boost the performance of your WordPress sites by using a caching layer (like ElastiCache for Memcached) and a content delivery network like Amazon CloudFront. *We will cover this later*

#### Task definition
Now that we have created the network, database, and shared storage, we are ready to deploy WordPress. We’ll use the [bitnami/wordpress] container image to create tasks. The task definition includes database credentials.

The WordPress container image is configured to store data at `/bitnami/wordpress`. As evident in the task definition above, the container mounts the EFS file system at `/bitnami/wordpress` on the container’s file system.

We also use an access point to access the EFS file system. EFS access points are application-specific entry points into an EFS file system that make it easier to manage application access to shared datasets. Access points can enforce a user identity, including the user’s POSIX groups, for all file system requests that are made through the access point. They can also enforce a different root directory for the file system so that clients can only access data in the specified directory or its sub-directories.

When running multiple WordPress installations, you can use a single EFS file system to persist data for multiple sites and isolate data by using an access point for each site.

Register the task definition:

```
WOF_TASK_DEFINITION_ARN=$(aws ecs register-task-definition \
--cli-input-json file://wp-task-definition.json \
--region $WOF_AWS_REGION \
--query taskDefinition.taskDefinitionArn --output text)
```


[//]: #
   [AWS CLI version 2]: <https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html>
   [AWS Management Console]: <https://console.aws.amazon.com/cloudformation/home>
   [bitnami/wordpress]: <https://hub.docker.com/r/bitnami/wordpress/dockerfile/>
