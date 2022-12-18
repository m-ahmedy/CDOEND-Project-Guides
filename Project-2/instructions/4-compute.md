# Building Highly Available Compute Power

A highly available compute on EC2 consists of the following resources:

- 3 Security Groups (Load Balancer, Web Servers, and Jumpbox Server)
- IAM Resources
  - 1 Role
  - 1 Policy
  - 1 Instance Profile
- Auto Scaling Resources
  - 1 Launch Configuration
  - 1 Auto-Scaling Group
  - 1 Auto-Scaling Policy
- Elastic Load Balancing Resources
  - 1 Target Group
  - 1 Application Load Balancer
  - 1 Listener
  - 1 Listener Rule
- (Optional) 2 Jumpbox (Bastion) Server

## Traffic firewall

All traffic paths are summarized in the below diagram

![](../assets/sg-traffic.jpeg)

### Jumpbox Security Group

Inbound Rules:

- Allow SSH Traffic on port 22 from anywhere

  Ideally the source should be limited to your own IP address only but for simplicity we are allowing traffic from anywhere

| Protocol | Port | Source    |
| -------- | ---- | --------- |
| TCP      | 22   | 0.0.0.0/0 |

Outbound rules: Unrestricted

### ALB Security Group

Inbound Rules:

- Allow HTTP Traffic on port 80 from anywhere

| Protocol | Port | Source    |
| -------- | ---- | --------- |
| TCP      | 80   | 0.0.0.0/0 |

Outbound Rules: Unrestricted

### Web Server Security Group

Inbound Rules:

- Allow HTTP Traffic on port 80 **only** from a resource having the **ALB Security Group**

  This allows our web service to be accessible from the Load Balancer only

- Allow SSH Traffic on port 22 **only** from a resource having the **Jumpbox Security Group**

  This makes SSH access to our web servers to be only allowed from the Jumpbox

| Protocol | Port | Source                 |
| -------- | ---- | ---------------------- |
| TCP      | 80   | ALB Security Group     |
| TCP      | 22   | Jumpbox Security Group |

So the final Security Groups definition would be like this

```yml
Parameters:
  # ...
  EnvironmentName:
    Description: The name of the deployment
    Type: String
    Default: My-HA-App

Resources:
  # ...
  JumpboxSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
    VpcId:
      Fn::ImportValue: !Sub "${EnvironmentName}-VPC"
    GroupDescription: Allow SSH from anywhere
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: 0.0.0.0/0

  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
    VpcId:
      Fn::ImportValue: !Sub "${EnvironmentName}-VPC"
    GroupDescription: Allow HTTP from anywhere, and HTTP to the Web Servers
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0

  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
    VpcId:
      Fn::ImportValue: !Sub "${EnvironmentName}-VPC"
    GroupDescription: Allow SSH from JumpBox and HTTP from the ALB
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        SourceSecurityGroupId: !Ref ALBSecurityGroup
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        SourceSecurityGroupId: !Ref JumpboxSecurityGroup
```

## Controlling Access with IAM

We would have an S3 bucket containing our static website files, and then we shall have a group of EC2 instances that would need to fetch these files

The bucket itself is not allowing public access, so we need to create IAM resources

Let's define the bucket name as a parameter in the `Parameters` section

```yml
Parameters:
  # ...
  BucketName:
    BucketName:
      Description: The bucket containing static website files
      Type: String
      Default: demo-website-667184564057
```

### A Role to access the bucket

First thing to do is to create a role that grants EC2 instances the required access to read the bucket

Required properties

- **Assume Role Policy Document**

  The trust policy that is associated with this role. Trust policies define which entities can assume the role.
  see [IAM role for applications that run on Amazon EC2 instances](https://docs.aws.amazon.com/autoscaling/ec2/userguide/us-iam-role.html) in the Amazon EC2 Auto Scaling User Guide

```yml
Resources:
  # ...
  EC2S3AccessRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: "Allow"
            Principal:
              Service:
                - "ec2.amazonaws.com"
            Action:
              - "sts:AssumeRole"
```

### The policy to grant access to the bucket

In [this](https://aws.amazon.com/premiumsupport/knowledge-center/s3-troubleshoot-copy-between-buckets/) troubleshooting guide for copying files between S3 buckets we can learn that the `sync` operation needs the following permissions

- Object level permissions: `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`
- Bucket level permissions: `s3:ListBucket`, `s3:GetBucketLocation`

The policy document can be something like this

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    }
  ]
}
```

_Note_: Since we only need the EC2 instances to fetch the files only and no editing is needed, so read only operations are needed

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    }
  ]
}
```

As for the policy resource, we need to define the following properties

- **Policy Name**

  The name of the policy document

- **Policy Document**

  The policy document

- **Roles**

  The name of the role to associate the policy with

```yml
Resources:
  # ...
  S3AccessPolicy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: S3AccessPolicy
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action:
              - s3:GetObject
            Resource: !Sub "arn:aws:s3:::${BucketName}/*"
          - Effect: Allow
            Action:
              - s3:ListBucket
              - s3:GetBucketLocation
            Resource: !Sub "arn:aws:s3:::${BucketName}"
      Roles:
        - !Ref EC2S3AccessRole
```

### The instance profile

An instance profile would be needed to map roles and grant their permissions to an EC2 instance

Properties needed:

- **Roles**

  The name of the role to associate with the instance profile

```yml
Resources:
  # ...
  WebServerInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref EC2S3AccessRole
```

## Auto Scaling

On top of the network infrastructure we deployed in the previous part, we need to deploy an Auto-Scaling group of our webserver instances spun across our two **private subnets**

The resources in this section are

- 1 Launch Configuration
- 1 Auto Scaling Group
- 1 Auto Scaling Policy

### The launch configuration

The launch configuration can be used by an Auto Scaling group to configure Amazon EC2 instances

Needed properties:

- **Image ID**

  The ID of the Amazon Machine Image (AMI) that was assigned during registration

  An Ubuntu Server AMI can be obtained from the AMI Catalog in the same region of deployment

  A more elegant way could be to query SSM to fetch the latest AMI of a specific [Ubuntu Release with its code-name](https://wiki.ubuntu.com/Releases), read more [here](https://ubuntu.com/server/docs/cloud-images/amazon-ec2)

- **Instance Type**

  Specifies the instance type of the EC2 instance

  A requirement is already set to use **t3.small** or better

- **Key Name**

  The name of the key pair for SSH access

  _Note_: This value is set for debugging only, after finishing the deployment and making sure everything works correctly we shall remove this property

- **Instance Monitoring**

  Controls whether instances in this group are launched with detailed (true) or basic (false) monitoring

  This property is needed for collecting metrics at a better frequency to allow for better ASG scaling reaction times

- **Security Groups**

  A list that contains the security groups to assign to the instances in the Auto Scaling group

- **IAM Instance Profile**

  The name or the Amazon Resource Name (ARN) of the instance profile associated with the IAM role for the instance

- **User Data** script:

  The **Base64-encoded** user data to make available to the launched EC2 instances

  Detailed explanation of all the steps

  | Step                                                                        | Implementation                                 |
  | --------------------------------------------------------------------------- | ---------------------------------------------- |
  | Declaring the executable                                                    | `#!/bin/bash`                                  |
  | Updating apt cache                                                          | `apt update`                                   |
  | Installing AWS CLI and Apache Web-Server                                    | `apt install -y apache2 awscli`                |
  | Starting Apache Service                                                     | `systemctl start apache2.service`              |
  | Enabling Apache Service at startup                                          | `systemctl enable apache2.service`             |
  | Syncing the contents of the serve folder with the contents of the S3 bucket | `aws s3 sync s3://${BucketName} /var/www/html` |

```yml
Parameters:
  # ...
  AmiId:
    Type: "AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>"
    Default: "/aws/service/canonical/ubuntu/server/jammy/stable/current/amd64/hvm/ebs-gp2/ami-id"

Resources:
  # ...
  WebServerASGLaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      ImageId: !Ref AmiID
      InstanceType: t3.small
      # KeyName: YOUR_KEY
      InstanceMonitoring: true
      SecurityGroups:
        - !Ref WebServerSecurityGroup
      IamInstanceProfile: !Ref InstanceProfile
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          apt update -y
          apt install apache2 awscli -y

          systemctl start apache2.service
          systemctl enable apache2.service

          aws s3 sync s3://${BucketName} /var/www/html
```

### The auto-scaling group (ASG)

The group of EC2 instances that are managed by the EC2 Autoscaling service

To enforce the replacement of all EC2 instances of this group if any change was made to the launch configuration, we add a replacement policy property

Needed properties:

- **Launch Configuration Name**

  The name of the launch configuration to use to launch instances

  Referencing the resource created in the last step

- **Max Size**

  The maximum size of the group

  We will specify this value as a parameter and reference its value here

- **Min Size**

  The minimum size of the group

  We will specify this value as a parameter and reference its value here

- **Desired Capacity**

  The desired capacity is the initial capacity of the Auto Scaling group at the time of its creation and the capacity it attempts to maintain

  We will specify this value as a parameter and reference its value here

- **VPC Zone Identifier**

  A list of subnet IDs for a virtual private cloud (VPC) where instances in the Auto Scaling group can be created

  We will import the values from the networking stack, remember that the ASG compute instances are required to launch in the private subnets

- **Target Group ARNs**

  The Amazon Resource Names (ARN) of the Elastic Load Balancing target groups to associate with the Auto Scaling group. Instances are registered as targets with the target groups

  Here we will reference the Target Group resource, which will be created later this guide

```yml
Parameters:
  # ...
  MinCapacity:
    Type: String
    Default: "2"

  MaxCapacity:
    Type: String
    Default: "4"

  DesiredCapacity:
    Type: String
    Default: "2"

Resources:
  # ...
  WebServerASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    UpdatePolicy:
      AutoScalingReplacingUpdate:
        WillReplace: true
    Properties:
      LaunchConfigurationName: !Ref WebServerASGLaunchConfiguration
      MaxSize: !Ref MaxCapacity
      MinSize: !Ref MinCapacity
      DesiredCapacity: !Ref DesiredCapacity

      VPCZoneIdentifier:
        - Fn::ImportValue: !Sub "${EnvironmentName}-Private-Subnet-1"
        - Fn::ImportValue: !Sub "${EnvironmentName}-Private-Subnet-2"

      TargetGroupARNs:
        - !Ref ELBTargetGroup # To be created later
```

### The auto-scaling policy

This is an optional resource, but we will create it anyway to explore the idea of implementing auto-scaling using CloudFormation

Needed properties:

- **Auto Scaling Group Name**

  The name of the Auto Scaling group, referencing the ASG resource

- **Policy Type**

  The type of the policy, here we will use the simplest auto-scaling policy type which is the Target Tracking

- **Target Tracking Configuration**

  - **Predefined Metric Specification**

    - **Predefined Metric Type**

      The metric type to be tracked, which for simplicity will be tracking the Average CPU Utilization of the ASG instances

  - **Target Value**

    The target value for the metric, here we will reference the value to be dynamically set in the stack as a parameter

```yml
Parameters:
  # ...
  CPUPolicyTargetValue:
    Type: Number
    Default: 30

Resources:
  # ...
  WebServerASGCPUScalingPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref WebServerASG
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: !Ref CPUPolicyTargetValue
```

## Elastic Load Balancing

The last pieces that are required to create a fully functional, highly available application is to add a load balancing feature

First we need to assign the ASG instances to a Target Group (done by referencing the TG in the ASG properties), A target group is used to route requests to one or more registered targets. When you create each listener rule, you specify a target group and conditions. When a rule condition is met, traffic is forwarded to the corresponding target group.

So in order to implement Load Balancing to our architecture we need the following

- 1 Target Group
- 1 Application Load Balancer
- 1 Listener
- 1 Listener Rule

_Note_: All of these resources are from the current generation of Load Balancers on AWS which is under the **ElasticLoadBalancingV2** resources category

Let's discuss each one

### Target Group

Specifies a target group for an Application Load Balancer, which will be created later on

Needed properties:

- **VPC ID**

  The identifier of the virtual private cloud (VPC) that the targets are residing in

  The value will be imported from the networks stack

- **Port, Protocol**

  The port and the protocol on which the targets receive traffic

  These values must match the ones specified on the targets, in our case they are 80 and HTTP, for Port and Protocol respectively

- **Health Checks**

  We can explicitly define health checks intervals and thresholds, but we will add them for documentation

```yml
Resources:
  # ...
  ELBTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      VpcId:
        Fn::ImportValue: !Sub "${EnvironmentName}-VPC"
      Port: 80
      Protocol: HTTP
      HealthCheckEnabled: true
      HealthCheckIntervalSeconds: 10
      HealthCheckPath: /
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 8
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 5
```

### The Application Load Balancer (ALB)

The load balancer that will be exposed to users and will balance the incoming traffic to targets

Needed properties:

- **Name**

  This optional, but it's always great to have a unique name for our customer facing resources

- **Subnets**

  The IDs of the public subnets, we will reference the two public subnets created in the networks stack

- **Security Groups**

  Application Load Balancers are VPC bound resources, so we need to define the IDs of the security groups for the load balancer

  We will reference the resource created earlier this stack for this purpose

```yml
Resources:
  # ...
  ELBLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub "${EnvironmentName}-ALB"
      Subnets:
        - Fn::ImportValue: !Sub "${EnvironmentName}-Public-Subnet-1"
        - Fn::ImportValue: !Sub "${EnvironmentName}-Public-Subnet-2"
      SecurityGroups:
        - !Ref LoadBalancerSecurityGroup
```

### The Listener

The listener is a process that runs on the load balancers and listens to traffic on a certain port and protocol, then it follows the rule that defines what to do with this incoming traffic

Needed properties:

- **Port, Protocol**

  These Port and Protocol are different from the ones that were defined in the Target Group resource, in this configuration they are directly associated with the customer facing requests on the load balancer, not the targets

- **Load Balancer ARN**

  The Amazon Resource Name (ARN) of the load balancer, referenced from the resource created earlier

- **Default Actions**

  A list of actions that will be used for dealing with traffic that does not follow any custom rule, we will define a simple default action

  - **Action Type**: Forward the traffic to a target group
  - **Target Group ARN**: The ARN of the target group that the traffic will be forwarded to

```yml
Resources:
  ELBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref ELBTargetGroup
      LoadBalancerArn: !Ref ELBLoadBalancer
      Port: 80
      Protocol: HTTP
```

### The listener rule

The rule which the listener will follow for certain condition

Needed properties:

- **Actions**

  The same as the default action defined on the listener, forwarding the traffic to the target group

- **Conditions**

  The conditions on which the listener will check on the incoming request and see if they are met

  We don't need to have fancy configuration for the condition, the only one we need is to match the path on the incoming HTTP request (`path-pattern`) to any pattern (`/*`)

- **Listener ARN**

  The Amazon Resource Name (ARN) of the listener, the resource created previously

- **Priority**

  The rule priority, this is a required value so will be set to 1 (or any number)

```yml
Resources:
  ELBListenerRule:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      Actions:
        - Type: forward
          TargetGroupArn: !Ref ELBTargetGroup
      Conditions:
        - Field: path-pattern
          Values:
            - "/*"
      ListenerArn: !Ref ELBListener
      Priority: 1
```

## The Jumpbox Servers

The Jumpbox server (or a bastion host) is a simple EC2 server that is residing on a public subnet, and allows the user to see and debug private resources from the outside the VPC

Needed properties:

- **Instance Type**

  The type of the instance, t2.micro is suitable because it's free tier eligible

- **Image ID**

  The ID of the AMI

- **Key Name**

  The SSH key pair name (Must be present on the region, can be specified with a parameter)

- **Security Group IDs**

  The ID of the security group that allows SSH traffic and is allowed to access private resources (Check the Security Groups definitions)

- **Subnet ID**

  Must be the ID of a public subnet

- **Name**

  To easily differentiate the instance we will give it a unique name tag

```yml
Parameters:
  # ...
  JumpboxKeyName:
    Type: String
    Default: proj2-jumpbox-key

Resources:
  # ...
  Jumpbox1:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: !Ref AmiID
      KeyName: !Ref JumpboxKeyName
      SecurityGroupIds:
        - !Ref JumpboxSecurityGroup
      SubnetId:
        Fn::ImportValue: !Sub "${EnvironmentName}-Public-Subnet-1"
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-Jumpbox-1"
  
  Jumpbox2:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: !Ref AmiID
      KeyName: !Ref JumpboxKeyName
      SecurityGroupIds:
        - !Ref JumpboxSecurityGroup
      SubnetId:
        Fn::ImportValue: !Sub "${EnvironmentName}-Public-Subnet-2"
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-Jumpbox-2"
```

## Outputs: ALB Domain Name and Jumpbox Hostnames

```yml
Outputs:
  Jumpbox1PublicHostname:
    Description: The Public IP Address of Jumpbox 1
    Value: !GetAtt Jumpbox1.PublicIp

  Jumpbox1PublicHostname:
    Description: The Public IP Address of Jumpbox 2
    Value: !GetAtt Jumpbox2.PublicIp

  LoadBalancerDNSName:
    Description: DNS Name of the web application
    Value: !Join
      - ""
      - - "http://"
        - !GetAtt ELBLoadBalancer.DNSName
    Export:
      Name: !Sub ${EnvironmentName}-ELB-DNS-Name
```
