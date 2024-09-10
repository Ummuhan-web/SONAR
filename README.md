# VPC CloudFormation Template

This CloudFormation template creates a Virtual Private Cloud (VPC) with a flexible and modular design for use across other stacks. It provisions a VPC along with one public, one private, and one database subnet, distributed across three Availability Zones (AZs) in a single AWS region. The VPC and its components are designed to be exported for use in other CloudFormation stacks, such as infrastructure templates, using AWS CloudFormation's `ImportValue` function.

## Architecture

The template builds a VPC with the following components:

### VPC
The main VPC where all subnets will reside, with a configurable CIDR block passed as a parameter. The VPC supports multiple subnet types, including public, private, and database subnets.

### Subnets

- **Public Subnets**: Three public subnets, one in each of the three AZs, where instances can be launched with public IPs to provide internet access.
- **Private Subnets**: Three private subnets, one in each AZ, where instances can communicate outbound to the internet through the NAT Gateway.
- **Database Subnets**: Three isolated subnets across AZs designed for use by database services such as Amazon RDS or Aurora. These subnets do not have direct internet access for security purposes.

### Internet Gateway and NAT Gateway

- **Internet Gateway (IGW)**: Provides internet access for public subnets, allowing inbound and outbound traffic to resources such as web servers.
- **NAT Gateway (NGW)**: Enables instances in the private subnets to access the internet for updates or other services while remaining inaccessible from the internet.

### Route Tables

- **Public Route Table**: Routes traffic from public subnets to the internet via the Internet Gateway.
- **Private Route Table**: Routes traffic from private subnets through the NAT Gateway for outbound internet access.
- **Database Route Table**: These route tables are dedicated to the database subnets, which are typically isolated from the internet unless explicitly configured otherwise.

### Outputs

Each of the following components is exported so that they can be referenced by other CloudFormation stacks using `Fn::ImportValue`:

- VPC ID
- Public Subnet IDs (across 3 AZs)
- Private Subnet IDs (across 3 AZs)
- Database Subnet IDs (across 3 AZs)

By using this VPC CloudFormation stack, other stacks (e.g., those with Application Load Balancers, EC2 instances, and Aurora) can import the VPC and subnet IDs, avoiding the need to duplicate network configuration.

---

### Example Use Case:

Once the VPC stack is created, other infrastructure stacks can import the VPC and subnet components and directly reference the exported values. This modular approach allows flexibility, reuse, and simplified management of your AWS infrastructure.


# Infrastructure CloudFormation Template

This CloudFormation template builds the infrastructure using the VPC created from a previous template. It imports the VPC, subnets, and necessary networking components and provisions the following:

- **Application Load Balancer (ALB)** in the public subnets.
- **Auto Scaling group** with EC2 instances located in different private subnets.
- EC2 instances serve a "Hello World" webpage.
- **Amazon Aurora** database in the DB subnets.
- **NAT Gateway** in the public subnet.

This template assumes that the **VPC**, **Public**, **Private**, and **DB subnets** have already been created in another stack and are exported using the `Export` function. We import them using the `Fn::ImportValue` function.

## Breakdown

### Imports:
The template imports the VPC and subnets (public, private, and DB) created by the previous stack using `Fn::ImportValue`.

### Application Load Balancer (ALB):
- ALB is provisioned across the three public subnets and listens on port 80 (HTTP).
- It forwards traffic to a target group containing the EC2 instances in the private subnets.

### Auto Scaling Group:
- EC2 instances are launched in three private subnets using an **Auto Scaling Group**.
- The instances are web servers that serve a simple "Hello World" HTML page.

### Amazon Aurora:
- An Amazon Aurora **DB Cluster** is created in the three DB subnets, and its security group allows traffic from the EC2 instances via port 3306 (MySQL).

### Security Groups:
- Separate security groups are configured for the ALB, EC2 instances, and Aurora DB.
- The EC2 security group allows HTTP traffic from the ALB, and the Aurora security group allows MySQL traffic from the EC2 instances.

## Reusability:
This template can be reused for additional Availability Zones by importing the respective subnets in the same region.





















