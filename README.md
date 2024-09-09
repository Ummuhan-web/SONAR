# AWS CloudFormation Template: VPC with Multi-AZ Subnets

## Overview

This template sets up a Virtual Private Cloud (VPC) in AWS with subnets distributed across multiple Availability Zones (AZs). Specifically, it creates:

- A VPC with a CIDR block of `10.0.0.0/16`
- Public, private, and database subnets in three Availability Zones

## Explanation

### VPC Creation

**`MyVPC`**:
- Defines the VPC with a CIDR block of `10.0.0.0/16`.

### Internet Gateway

**`MyInternetGateway`**:
- Provides internet access to the public subnets.

### Subnets

- **Public Subnets**:
  - **`MyPublicSubnet1`, `MyPublicSubnet2`, `MyPublicSubnet3`**: These subnets are configured to provide internet access. They are typically used for resources that need to be accessible from the internet, such as web servers.

- **Private Subnets**:
  - **`MyPrivateSubnet1`, `MyPrivateSubnet2`, `MyPrivateSubnet3`**: These subnets are used for resources that do not need direct internet access, such as application servers. They are isolated from the internet but can access resources in the public subnets if necessary.

- **DB Subnets**:
  - **`MyDBSubnet1`, `MyDBSubnet2`, `MyDBSubnet3`**: Dedicated for database instances. These subnets are designed to host databases that need to be isolated from both the public and private subnets, providing additional security.

### Route Table and Associations

- **`MyRouteTable`**:
  - Configures routing for the VPC. It determines how traffic is directed within the VPC and to/from the internet.

- **`MyPublicRoute`**:
  - Routes traffic to the internet via the Internet Gateway. This route is associated with the public subnets to enable internet access.

- **Subnet Associations**:
  - Associates the public subnets with the route table. This allows instances in these subnets to communicate with the internet.

### Outputs

- Provides IDs for the VPC and subnets (public, private, and DB). These outputs are useful for referencing in other CloudFormation stacks or configurations, allowing you to easily integrate this VPC setup with other AWS resources.

## Deployment

To deploy this CloudFormation template:

1. Save the YAML template as `vpc-stack.yaml`.
2. Use the AWS CloudFormation console or AWS CLI to create a stack.

### Deploy Using CloudFormation Console

- Go to the AWS CloudFormation console.
- Create a new stack and upload the `vpc-stack.yaml` file.
- Specify the parameters such as `AvailabilityZones`.

### Deploy Using AWS CLI

```sh
aws cloudformation create-stack --stack-name my-vpc-stack --template-body file://vpc-stack.yaml --parameters ParameterKey=AvailabilityZones,ParameterValue="us-east-1a,us-east-1b,us-east-1c"

```
# VPC Stack Deployment

## Overview

To use `vpc-stack.yaml` in another CloudFormation template, create a parent stack that references the VPC stack as a nested stack.

## Configuration

### Nested Stack Reference

- **Type:** `AWS::CloudFormation::Stack`
- **TemplateURL:** Point to the location of `vpc-stack.yaml` in S3. Upload `vpc-stack.yaml` to an S3 bucket and replace the URL accordingly.

### Outputs

- Retrieves outputs from the nested VPC stack to make them available in the parent stack.

## Deployment Steps

### 1. Upload the VPC Stack Template to S3

- Upload `vpc-stack.yaml` to an S3 bucket.

### 2. Deploy the Parent Stack

- Save the parent template as `parent-stack.yaml`.

#### Using the AWS CloudFormation Console

1. Open the AWS CloudFormation console.
2. Create a new stack and upload the `parent-stack.yaml` file.
3. Specify the parameters for the VPC stack.

#### Using AWS CLI

```sh
aws cloudformation create-stack --stack-name my-parent-stack --template-body file://parent-stack.yaml --parameters ParameterKey=VPCName,ParameterValue=my-vpc ParameterKey=CIDRBlock,ParameterValue=10.0.0.0/16 ParameterKey=AvailabilityZones,ParameterValue='us-east-1a,us-east-1b,us-east-1c'


# CloudFormation Template for ALB, Auto Scaling, EC2, and Amazon Aurora DB

This CloudFormation template provisions an infrastructure across **three Availability Zones (AZs)**, including:

- An Application Load Balancer (ALB) in public subnets.
- Auto Scaling for EC2 instances in public subnets that serve a "Hello World" webpage.
- Amazon Aurora DB instances in private DB subnets.
- VPC components (such as subnets and route tables) imported from a parent CloudFormation stack.

## Architecture Overview

- **Application Load Balancer (ALB)**: The ALB is deployed in public subnets across three AZs to distribute traffic to the EC2 instances.
- **Auto Scaling EC2 Instances**: EC2 instances are launched and scaled in public subnets to meet traffic demands.
- **Amazon Aurora DB**: The Aurora DB instances are hosted in private DB subnets to ensure secure communication between the web server and database.
- **VPC Components**: All network components such as VPC, subnets, and route tables are imported from a parent stack stored in S3.

## Template Breakdown

### VPC Import
The VPC and its related subnets are imported from the parent stack located at `https://s3.amazonaws.com/my-bucket/vpc-stack.yaml`. This includes public, private, and DB subnets spread across three availability zones.

### ALB & Auto Scaling
- The ALB is set up in three public subnets, one in each AZ, to balance traffic across multiple EC2 instances.
- An Auto Scaling group is configured to manage the number of EC2 instances in the public subnets, based on the incoming traffic.

### EC2 Instance
- EC2 instances are automatically launched in the public subnets via Auto Scaling and serve a "Hello World" webpage using Apache.
- These EC2 instances can securely communicate with the Amazon Aurora DB, which resides in private DB subnets.

### Amazon Aurora DB
- An Aurora DB cluster is deployed across private DB subnets in all three availability zones.
- The security group configuration allows secure communication between the EC2 instances (in the public subnets) and the DB (in the private subnets).

### Security Groups
- A security group is applied to the ALB, allowing HTTP traffic from the public internet (port 80).
- Another security group is applied to the EC2 instances and the Amazon Aurora DB, allowing MySQL traffic between them over port 3306.

### Subnet Configuration
The template supports three availability zones, each with distinct subnets for public, private, and database traffic. This ensures traffic isolation and better routing efficiency.

## Customization

- **Amazon Linux 2 AMI**: Replace the AMI ID `ami-0c55b159cbfafe1f0` with the latest Amazon Linux 2 AMI for your region.
- **EC2 Instance Type**: Adjust the EC2 instance type (`t2.micro` by default) as per your requirements.
- **Key Pair**: Replace the key pair with your own for SSH access to the EC2 instances.
- **Database Credentials**: Set the appropriate values for the Aurora DB master username and password.
- **VPC Import**: Ensure that the parent VPC stack provides the necessary public, private, and DB subnets across all three AZs.

## Conclusion

This CloudFormation template is reusable and flexible for any three-AZ architecture with separate subnets for public, private, and database traffic. It ensures high availability and fault tolerance by distributing resources across multiple availability zones.


