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
      Scheme: internet-facing
      LoadBalancerAttributes:
        - Key: idle_timeout.timeout_seconds
          Value: '60'
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Type: application 
```

### 2. **Auto Scaling Group (ASG)**
The Auto Scaling Group maintains the appropriate number of EC2 instances across AZs. It automatically adjusts the number of instances based on demand and replaces unhealthy instances in case of failure.

```yaml
  AutoScalingGroup:
    Type: 'AWS::AutoScaling::AutoScalingGroup'
    Properties:
      VPCZoneIdentifier:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
        - !Ref PrivateSubnet3
      LaunchConfigurationName: !Ref AutoScalingLaunchConfig
      MinSize: 3
      MaxSize: 3
      DesiredCapacity: 3
      TargetGroupARNs:
        - !Ref ALBTargetGroup 
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
  AuroraDBCluster:
    Type: 'AWS::RDS::DBCluster'
    Properties:
      Engine: aurora
      EngineMode: provisioned
      MasterUsername: !Sub '{{resolve:ssm-secure:/myapp/db/masteruser:1}}'
      MasterUserPassword: !Sub '{{resolve:ssm-secure:/myapp/db/masterpassword:1}}'
      DBSubnetGroupName: !Ref DBSubnetGroup
      VpcSecurityGroupIds:
        - !Ref DBSecurityGroup
      BackupRetentionPeriod: 
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


