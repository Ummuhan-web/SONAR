# Disaster Recovery Plan

## Objective
The objective of this Disaster Recovery Plan is to ensure service continuity, resilience, and quick recovery in the event of a system failure. The plan ensures that our infrastructure remains highly available across multiple Availability Zones (AZs) using AWS services, including Auto Scaling, Route 53, NAT Gateways, Aurora, and Application Load Balancers (ALB).

## Strategy
The disaster recovery strategy includes:
- **Multi-AZ Redundancy**: Deploy resources across multiple AZs for high availability.
- **Auto Scaling**: Automatically recover from failures by scaling EC2 instances.
- **Route 53 Failover**: Redirect traffic to healthy resources in case of failure (this is a crucial part of the infrastructure, therefore added to the plan although it was not required in the assessment).
- **NAT Gateway Per AZ**: Ensure private subnet access to the internet, even in case of AZ failure.
- **Aurora Replica Promotion**: Promote Aurora database replicas during primary database failure.
- **Health Checks**: Continuously monitor the health of the infrastructure components.

## Key Components

### 1. **Application Load Balancer (ALB)**
The ALB distributes incoming traffic across multiple AZs, ensuring high availability. It routes traffic only to healthy targets when there's failover in an AZ.

```yaml
  ALB:
    Type: 'AWS::ElasticLoadBalancingV2::LoadBalancer'
    Properties:
      Name: MyALB
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
        - !Ref PublicSubnet3
      SecurityGroups: [ !Ref ALBSecurityGroup ]
      Scheme: internet-facing
      LoadBalancerAttributes:
        - Key: idle_timeout.timeout_seconds
          Value: '60'
      Tags:
        - Key: Name
          Value: MyALB
```

### 2. **Auto Scaling Group (ASG)**
The Auto Scaling Group maintains the appropriate number of EC2 instances across AZs. It automatically adjusts the number of instances based on demand and replaces unhealthy instances in case of failure.

```yaml
  AutoScalingGroup:
    Type: 'AWS::AutoScaling::AutoScalingGroup'
    Properties:
      LaunchConfigurationName: !Ref LaunchConfiguration
      MinSize: '1'
      MaxSize: '3'
      DesiredCapacity: '2'
      VPCZoneIdentifier:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
        - !Ref PrivateSubnet3
      Tags:
        - Key: Name
          Value: WebServer
          PropagateAtLaunch: true
   
```

### 3. **Route 53 Failover Routing**
Route 53 provides DNS-based failover routing to ensure that traffic is directed to healthy resources. It monitors the health of the primary resources and automatically re-routes traffic to a backup resource if the primary resource fails.


### 4. **NAT Gateway Per AZ**
Deploying a NAT Gateway in each Availability Zone ensures that resources in private subnets maintain internet access even if an AZ fails. Each private subnet is associated with a route table that points to the NAT Gateway in its respective AZ.

```yaml
  NATGateway1:
    Type: 'AWS::EC2::NatGateway'
    Properties:
      AllocationId: !GetAtt EIP1.AllocationId
      SubnetId: !Ref PublicSubnet1

  NATGateway2:
    Type: 'AWS::EC2::NatGateway'
    Properties:
      AllocationId: !GetAtt EIP2.AllocationId
      SubnetId: !Ref PublicSubnet2

  NATGateway3:
    Type: 'AWS::EC2::NatGateway'
    Properties:
      AllocationId: !GetAtt EIP3.AllocationId
      SubnetId: !Ref PublicSubnet3 
```


### 5. Aurora Replica Promotion**
Amazon Aurora automatically promotes a read replica to be the new primary instance with write capability in the event of a primary instance failure. This process is automated and typically completed within seconds to ensure minimal downtime.

```yaml
  AuroraCluster:
    Type: 'AWS::RDS::DBCluster'
    Properties:
      Engine: aurora-mysql
      EngineVersion: '8.0.mysql_aurora.3.05.2'
      MasterUsername: !Ref MasterUsername
      MasterUserPassword: !Ref MasterUserPassword
      DBSubnetGroupName: !Ref DBSubnetGroup
      VpcSecurityGroupIds: [ !Ref DBClusterSecurityGroup ]
      BackupRetentionPeriod: 7
      Tags:
        - Key: Name
          Value: MyAuroraCluster
  ```

### 6. **Health Checks**
Health checks ensure that the Load Balancer and instances are operational. Route 53 uses health checks to determine the health of endpoints and Route 53 DNS records, while ALB performs health checks on targets within the Auto Scaling Group.

#### Health Check for Route 53
Route 53 health checks monitor the availability and performance of resources, directing traffic to healthy endpoints.

#### Health Check for Application Load Balancer (ALB)
The ALB performs health checks on the instances registered with the Target Group, ensuring only healthy instances receive traffic.

## Further Considerations for DR
By considering the following additional factors, we can further strengthen our disaster recovery strategy and improve the resilience of our infrastructure. These will be dependent on our cost considerations and the extent of our customer outreach requirements. 

### Cross-Region Disaster Recovery
- **Backup Infrastructure**: Consider setting up a backup infrastructure in a separate AWS region to enhance disaster recovery capabilities. This includes replicating key resources and data to ensure business continuity in case of a regional outage.
- **Route 53 Failover Routing**: Utilize Route 53’s failover routing policies to direct traffic to a secondary region if the primary region experiences issues.
- **S3 Cross-Region Replication**: Implement cross-region replication for S3 buckets to ensure that your data is available in multiple regions, providing resilience against region-wide failures.

### Data Backup & Restore
- **Regular Backups**: Schedule regular backups for critical resources such as Amazon Aurora databases. Ensure that these backups are stored in a separate region to protect against data loss.
- **Snapshots**: Maintain snapshots of essential data and configurations. Store these snapshots in a different region to facilitate quick restoration in the event of a regional failure.

### Testing the DR Plan
- **Simulate Failures**: Regularly test the disaster recovery plan by simulating various failure scenarios such as Availability Zone (AZ) failure, instance failure, or database failure. This ensures that all components of the DR plan function as expected.
- **Review and Improve**: After each test, review the outcomes and refine the DR plan based on findings. Ensure that all team members are familiar with their roles and responsibilities during a disaster.

# Incorporating Amazon S3 into CloudFormation for Disaster Recovery

Incorporating Amazon S3 into your CloudFormation infrastructure for disaster recovery (DR) can enhance the resilience of your architecture. Below are several ways S3 can be integrated into your infrastructure template to support a Disaster Recovery Plan (DRP):

## 1. S3 as a Backup and Restore Mechanism
Store periodic backups of critical data, application configurations, logs, or even database snapshots in S3. You can set up automated backups from Amazon EC2 or Aurora database clusters to S3 using AWS Lambda or AWS Backup service. These backups can be regularly restored in case of an emergency.

### How to Implement:
- Use S3 buckets to store EC2 AMI snapshots, Aurora database backups, or Auto Scaling logs.
- Include backup configuration for Aurora databases or other critical components in the template.

## 2. Storing Infrastructure Configurations
Keep the CloudFormation templates themselves, as well as other configuration files (e.g., Lambda code or user-data scripts) stored in S3. This allows for quick re-deployment if the infrastructure needs to be rebuilt after a disaster.

### How to Implement:
- Store your infrastructure templates in a versioned S3 bucket.
- Use S3-backed AWS Lambda functions to automate disaster recovery processes such as automated failovers or environment rebuilds.

## 3. Cross-Region Replication (Optional for Multi-Region DR)
For multi-region DR plans, you can use S3 Cross-Region Replication (CRR) to automatically replicate your data from the primary region to a secondary region. This can be useful if your disaster recovery plan requires you to recover in a different region.

## 4. Storing Application Logs for Recovery
Store application logs (from EC2 instances or ALB access logs) in S3 for forensics and post-mortem analysis after a disaster event. These logs can help diagnose root causes or assist in system recovery.

### How to Implement:
- Configure EC2 Auto Scaling group to send logs to an S3 bucket.
- Enable S3 logging for your ALB.

## 5. S3 for Static Website Failover (with CloudFront)
Use S3 to serve as a failover mechanism for a static website. If your application hosted on EC2 or Aurora becomes unavailable, you can configure CloudFront to redirect to a static S3 website as a fallback.

### How to Implement:
- Create an S3 bucket configured as a static website.
- Use Route53 DNS failover with CloudFront to redirect traffic during a failure.

## 6. Automated Disaster Recovery Scripts Stored in S3
Store automated DR scripts or Lambda functions in S3. These can help automate the process of infrastructure recovery, such as launching replacement EC2 instances, restoring databases, or rerouting traffic.

### How to Implement:
- Store recovery scripts (e.g., shell scripts or CloudFormation templates) in an S3 bucket.
- Trigger these scripts via Lambda or AWS Systems Manager when a disaster strikes.

## 7. S3 Lifecycle Management for Cost Optimization
Use lifecycle policies to automatically move data to S3 Glacier or delete data after a set period, helping you maintain long-term backups for disaster recovery while optimizing costs.

## Summary of S3 Disaster Recovery Benefits

- **Data Durability:** S3 ensures 99.999999999% durability, making it highly reliable for storing critical backups.
- **Automated Failover:** With CloudFront and Route 53, you can create automated failover scenarios using S3 to ensure continuity.
- **Cost-Effective Storage:** Utilize different storage classes, such as Standard and Glacier, to optimize costs while maintaining backups.
- **Easy Restoration:** Data stored in S3 can be easily restored, allowing you to quickly rebuild your infrastructure if needed.
- **Resilience with Cross-Region Replication (Optional):** S3 replication can enhance disaster recovery by ensuring data availability across multiple regions.

By integrating S3 into your CloudFormation infrastructure template, you can create a robust and scalable disaster recovery (DR) plan.












