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



#### Run WordPress
we’ll create an ECS cluster to run WordPress and create an ECS service that maintains two WordPress replicas for high availability. It also integrates with the ALB and automatically updates ALB targets as WordPress tasks are created and destroyed.

##### Create an ECS cluster:

```
aws ecs create-cluster \
  --cluster-name $WOF_ECS_CLUSTER_NAME \
  --region $WOF_AWS_REGION
```

###### Create a security group that accepts traffic on port 8080 from the ALB’s security group:

```
WOF_SVC_SG_ID=$(aws ec2 create-security-group \
  --description Svc-WordPress-on-Fargate \
  --group-name Svc-WordPress-on-Fargate \
  --vpc-id $WOF_VPC_ID --region $WOF_AWS_REGION \
  --query 'GroupId' --output text)
```

##### Accept traffic on port 8080

```
aws ec2 authorize-security-group-ingress \
  --group-id $WOF_SVC_SG_ID --protocol tcp \
  --port 8080 --source-group $WOF_ALB_SG_ID \
  --region $WOF_AWS_REGION
```

##### Create an ECS service:

```
aws ecs create-service \
  --cluster $WOF_ECS_CLUSTER_NAME \
  --service-name wof-efs-rw-service \
  --task-definition "${WOF_TASK_DEFINITION_ARN}" \
  --load-balancers targetGroupArn="${WOF_TG_ARN}",containerName=wordpress,containerPort=8080 \
  --desired-count 2 \
  --platform-version 1.4.0 \
  --launch-type FARGATE \
  --deployment-configuration maximumPercent=100,minimumHealthyPercent=0 \
  --network-configuration "awsvpcConfiguration={subnets=["$WOF_PRIVATE_SUBNET0,$WOF_PRIVATE_SUBNET1"],securityGroups=["$WOF_SVC_SG_ID"],assignPublicIp=DISABLED}"\
  --region $WOF_AWS_REGION
```

###### Wait until there two running tasks

```
watch aws ecs describe-services \
  --services wof-efs-rw-service \
  --cluster $WOF_ECS_CLUSTER_NAME \
  --region $WOF_AWS_REGION \
  --query 'services[].runningCount' 
```

Once two tasks are running you can proceed to the next steps.

#### Finally, Access WordPress admin dashboard
Once the Fargate tasks are running, log into the WordPress admin dashboard. Obtain the dashboard address:

```
echo "http://$(aws elbv2 describe-load-balancers \
  --names wof-load-balancer --region $WOF_AWS_REGION \
  --query 'LoadBalancers[].DNSName' --output text)/wp-admin/"
```

The admin username is `user` and the password is `bitnami`. Please change the password as soon as possible.

#### Test data persistence
Now that the WordPress installation is complete, it’s time to test data persistence. While in your WordPress admin dashboard, make a site configuration change like activating a new site theme. Once you’ve made the change, terminate all WordPress containers:

```
aws ecs update-service \
  --cluster $WOF_ECS_CLUSTER_NAME \
  --region $WOF_AWS_REGION \
  --service wof-efs-rw-service \
  --task-definition "$WOF_TASK_DEFINITION_ARN" \
  --desired-count 0 
```

Wait until there are no active replicas. You can use watch to get the replica count:

```
watch aws ecs describe-services \
  --services wof-efs-rw-service \
  --cluster $WOF_ECS_CLUSTER_NAME \
  --region $WOF_AWS_REGION \
  --query 'services[].runningCount'
```

Once all tasks have been terminated, scale the service back to a minimum of three replicas:

```
aws ecs update-service \
  --cluster $WOF_ECS_CLUSTER_NAME \
  --region $WOF_AWS_REGION \
  --service wof-efs-rw-service \
  --task-definition "$WOF_TASK_DEFINITION_ARN" \
  --desired-count 2
```

When the tasks are running, go to the WordPress admin dashboard, and verify that your changes have persisted.

#### Service Auto Scaling
ECS Service Auto Scaling helps you maintain the level of performance for your application as its load waxes and wanes.

Here’s an example that demonstrates setting Service Auto Scaling for WordPress. First, we’ll register the WordPress ECS service with Application Auto Scaling, then we’ll create a policy that automatically adjusts the task replica count based on the service’s average CPU utilization.

###### Register the ECS service with Application Auto Scaling:

```
aws application-autoscaling \
  register-scalable-target \
  --region $WOF_AWS_REGION \
  --service-namespace ecs \
  --resource-id service/${WOF_ECS_CLUSTER_NAME}/wof-efs-rw-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 4
```

##### Configure Application Auto Scaling policy:

```
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/${WOF_ECS_CLUSTER_NAME}/wof-efs-rw-service \
  --policy-name cpu75-target-tracking-scaling-policy \
  --policy-type TargetTrackingScaling \
  --region $WOF_AWS_REGION \
  --target-tracking-scaling-policy-configuration file://scaling.config.json
```

When the WordPress service’s average CPU utilization rises above 75%, a scale-out alarm triggers Service Auto Scaling to add another task to the WordPress service to decrease the load on the running tasks and ensure that users don’t experience a service disruption. Conversely, when the average CPU utilization dips below 75%, a scale-in alarm triggers a decrease in the service’s desired count.

Now, you can use any utility generate load, and test the auto scaling configuration:

Here i'm using the apache benchmark tool, and while it will give you a solid idea of some performance, it is a bad idea to only depend on it if you plan to have your site exposed to serious stress in production.

Having said that, here's the most common and simplest parameters:

- -c: ("Concurrency"). Indicates how many clients (people/users) will be hitting the site at the same time. While ab runs, there will be -c clients hitting the site. This is what actually decides the amount of stress your site will suffer during the benchmark.

- -n: Indicates how many requests are going to be made. This just decides the length of the benchmark. A high -n value with a -c value that your server can support is a good idea to ensure that things don't break under sustained stress: it's not the same to support stress for 5 seconds than for 5 hours.

- -k: This does the "KeepAlive" funcionality browsers do by nature. You don't need to pass a value for -k as it it "boolean" (meaning: it indicates that you desire for your test to use the Keep Alive header from HTTP and sustain the connection). Since browsers do this and you're likely to want to simulate the stress and flow that your site will have from browsers, it is recommended you do a benchmark with this.

The final argument is simply the host. By default it will hit http:// protocol if you don't specify it.

``` ab -k -c 350 -n 20000 <worpress URL> ```



[//]: #
   [AWS CLI version 2]: <https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html>
   [AWS Management Console]: <https://console.aws.amazon.com/cloudformation/home>
   [bitnami/wordpress]: <https://hub.docker.com/r/bitnami/wordpress/dockerfile/>
