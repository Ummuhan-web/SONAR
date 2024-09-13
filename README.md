# CloudFormation Template for Highly Available VPC with Public, Private, and Database Subnets Across 3 AZs

## Overview

This CloudFormation template creates a highly available Virtual Private Cloud (VPC) architecture. It includes public, private, and database subnets spread across three Availability Zones (AZs) in a single AWS region. The stack also provisions essential resources like an Application Load Balancer (ALB), Auto Scaling Group, NAT Gateways, and an Amazon Aurora database.

## Key Components

### Parameters

- **VPCCIDR**: CIDR block for the VPC (default: `10.0.0.0/16`).
- **PublicSubnet1CIDR - DBSubnet3CIDR**: CIDR blocks for public, private, and database subnets in three AZs.
- **AMIId**: Amazon Machine Image (AMI) ID for EC2 instances.
- **InstanceType**: EC2 instance type (default: `t2.micro`).
- **KeyPairName**: SSH key pair for EC2 instances.
- **MasterUsername & MasterUserPassword**: Securely stored credentials in AWS Systems Manager (SSM) Parameter Store for the Aurora DB cluster.

### VPC

- **VPC**: The core network component, using a user-defined CIDR block to create a custom VPC.
- **Subnets**: 
  - **Public Subnets**: Provide internet-facing resources with public IP addresses.
  - **Private Subnets**: Hold resources not requiring direct internet access.
  - **Database Subnets**: Specifically for hosting the Aurora database instances, spread across AZs for high availability.

### Internet Gateway and NAT Gateways

- **Internet Gateway**: Attached to the VPC to allow public subnets internet access.
- **NAT Gateways**: Enable resources in private subnets to connect to the internet for updates, without exposing them to inbound internet traffic.

### Route Tables

- **Public Route Table**: Routes public traffic (0.0.0.0/0) to the Internet Gateway.
- **Private Route Table**: Routes private subnet traffic through NAT gateways, allowing internet access while remaining private.

### Amazon Aurora Cluster

- **Aurora DB Cluster**: Multi-AZ Amazon Aurora MySQL cluster with automated backups and a retention period of 7 days.
- **DB Subnet Group**: Aurora subnets defined across three AZs for high availability.
- **DBClusterSecurityGroup**: Secures Aurora DB, allowing traffic on port 3306 (MySQL).

### Security Groups

- **ALB Security Group**: Allows inbound HTTP traffic (port 80) from any IP address to the ALB.
- **Web Server Security Group**: Allows traffic from the ALB to EC2 instances on port 80, and allows the EC2 instances to access the Aurora DB on port 3306.
- **DBClusterSecurityGroup**: Restricts access to the Aurora DB to connections from internal private subnets.

### Application Load Balancer (ALB)

- **ALB**: Internet-facing load balancer that distributes HTTP traffic across the EC2 instances.
- **ALB Target Group**: Health checks and routes traffic to healthy EC2 instances in the Auto Scaling Group.

### Auto Scaling Group

- **Launch Configuration**: Defines how EC2 instances are launched, including instance type, AMI ID, security group, and user data for instance initialization.
- **Auto Scaling Group**: Ensures that a minimum of 1 and a maximum of 3 EC2 instances are running in private subnets, scaling based on load.

### Outputs

The stack exports the following resources for future reference or use by other CloudFormation stacks:
- **VPCID**: The ID of the created VPC.
- **Public Subnet IDs**: IDs for public subnets in each AZ.
- **Private Subnet IDs**: IDs for private subnets in each AZ.
- **DB Subnet IDs**: IDs for database subnets in each AZ.
- **ALB ARN**: The Amazon Resource Name (ARN) for the ALB.
- **Aurora Cluster Endpoint**: The connection endpoint for the Aurora DB cluster.

## Usage

1. Launch the CloudFormation stack using this template and provide the required parameters (e.g., AMI ID, EC2 key pair name, SSM parameter paths for the Aurora DB credentials).
2. Once deployed, the infrastructure will be fully set up with a scalable, highly available environment, ready to host a web application behind an ALB and securely connected to an Aurora DB.

### Notes

- **CIDR Blocks**: These are IP ranges used for defining network segments (subnets). CIDR blocks in this template control how traffic is routed within the VPC and between resources.
- **High Availability**: The use of multiple Availability Zones (AZs) ensures redundancy and fault tolerance.
- **Security**: Security groups and SSM parameters ensure secure access control for instances, databases, and application layers.

