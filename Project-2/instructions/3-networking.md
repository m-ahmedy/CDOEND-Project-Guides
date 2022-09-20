# Building Highly Available Network

A highly available network proposal consists of the following resources:

- 1 VPC
- 4 Subnets (2 Public, 2 Private)
- 1 Internet Gateway (attached to the VPC)
- 2 NAT Gateways + 2 Elastic IPs
- 1 Route Table for all Public Subnets + 1 Route Table **for each** Private Subnet

## CIDR Blocks

Using one of the [private CIDR Blocks](https://en.wikipedia.org/wiki/Private_network#Private_IPv4_addresses) specified by the IANA, one proposal is to use the following

We will go with 10.0.0.0/16 CIDR Block for the VPC, each subnet will take a portion of this block

Check this valuable resource for understanding [IPv4, CIDR, and VPC Subnets](https://www.youtube.com/watch?v=z07HTSzzp3o)

| Item             | CIDR        | Parameter Name     |
| ---------------- | ----------- | ------------------ |
| VPC              | 10.0.0.0/16 | VPCCIDR            |
| Private Subnet 1 | 10.0.0.0/24 | PrivateSubnet1CIDR |
| Public Subnet 1  | 10.0.1.0/24 | PublicSubnet1CIDR  |
| Private Subnet 2 | 10.0.2.0/24 | PrivateSubnet2CIDR |
| Public Subnet 2  | 10.0.3.0/24 | PublicSubnet2CIDR  |

So the `Parameters` section could be something like this

```yml
Parameters:
  VPCCIDR:
    Description: CIDR Block of the VPC
    Type: String
    Default: 10.0.0.0/16

  PrivateSubnet1CIDR:
    Description: CIDR Block of the first private subnet
    Type: String
    Default: 10.0.0.0/24

  PublicSubnet1CIDR:
    Description: CIDR Block of the first public subnet
    Type: String
    Default: 10.0.1.0/24

  PrivateSubnet2CIDR:
    Description: CIDR Block of the second private subnet
    Type: String
    Default: 10.0.2.0/24

  PublicSubnet2CIDR:
    Description: CIDR Block of the second public subnet
    Type: String
    Default: 10.0.3.0/24
```

## Base Resources

### VPC

The VPC Resource should have the following properties defined

- **CIDR Block**

  The IPv4 network range for the VPC, in CIDR notation

- **DNS Support**

  Indicates whether the DNS resolution is supported for the VPC. If enabled, queries to the Amazon provided DNS server

- **DNS Hostnames**

  Indicates whether the instances launched in the VPC get DNS hostnames. If enabled, instances in the VPC get DNS hostnames

```yml
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VPCCIDR
      EnableDnsSupport: true
      EnableDnsHostnames: true
```

### Subnets

All the subnets should have the following properties defined

- **VPC ID**

  The ID of the VPC the subnet is in

- **CIDR Block**

  The IPv4 CIDR block assigned to the subnet

- **Map public IP on launch**

  Indicates whether instances launched in this subnet receive a public IPv4 address, must be **enabled** for **Public Subnets**

- **Availability Zone**

  The Availability Zone of the subnet

  To set the Availability Zone, we can use [GetAZs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-getavailabilityzones.html) in conjunction with [Select](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-select.html) intrinsic functions

  Specifying an empty string is equivalent to specifying `AWS::Region`, both are valid to get the AZs of the region in which the stack is created

**Private Subnets**

```yml
Resources:
  # ...
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PrivateSubnet1CIDR
      AvailabilityZone: !Select [0, !GetAZs ""]

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PrivateSubnet2CIDR
      AvailabilityZone: !Select [1, !GetAZs ""]
```

**Public Subnets**

```yml
Resources:
  # ...
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnet1CIDR
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select [0, !GetAZs ""]

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnet2CIDR
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select [1, !GetAZs ""]
```

### Internet Gateway

An internet gateway is a horizontally scaled, redundant, and highly available VPC component that allows communication between your VPC and the internet

An internet gateway enables resources in your public subnets (such as EC2 instances) to connect to the internet if the resource has a public IPv4 address or an IPv6 address. Similarly, resources on the internet can initiate a connection to resources in your subnet using the public IPv4 address or IPv6 address

No properties have to be set in this resource

```yml
Resources:
  # ...
  InternetGateway:
    Type: AWS::EC2::InternetGateway
```

### VPC Gateway Attachment

After creating the Internet gateway, you then attach it to our VPC

- **VPC ID**

  The ID of the VPC

- **Internet Gateway ID**

  The ID of the internet gateway

```yml
Resources:
  # ...
  InternetGatewayVPCAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC
```

### Elastic IPs and NAT Gateways

NAT Gateways will be needed to allow private resources to get to the internet (and not vice-versa)

NAT Gateways should be created in public subnets for this use case, and that requires us to create an Elastic IP reservation for each NAT Gateway

For the EIP, there are no properties that need to be defined

As for the NAT Gateways, there are some properties that need definition

- **Allocation ID**

  The allocation ID of the Elastic IP address that's associated with the public NAT gateway

  To get the allocation ID of an EIP we use the `GetAtt` intrinsic function

- **Subnet ID**

  The ID of the subnet in which the NAT gateway is located

```yml
Resources:
  # ...
  EIP1:
    Type: AWS::EC2::EIP

  NatGateway1:
    Type: AWS::EC2::NatGateway
    Properties:
      SubnetId: !Ref PublicSubnet1
      AllocationId: !GetAtt EIP1.AllocationId

  EIP2:
    Type: AWS::EC2::EIP

  NatGateway2:
    Type: AWS::EC2::NatGateway
    Properties:
      SubnetId: !Ref PublicSubnet2
      AllocationId: !GetAtt EIP2.AllocationId
```

## Routing traffic

There are two kinds of traffic that will flow through the VPC

- Traffic that's targeted for a local resource within the VPC
- Other traffic that's targeted for a resource outside the VPC (to the internet)

And we have two different behaviors for traffic handling of resources, depending on the type of subnet they reside in

We have resources residing in Public Subnets

- Each resource has a **globally unique** **public IP address** for outside communication with the internet, as well as the private IP address for local communication within the VPC

- **Outbound**: Can access the internet directly, the source IP address is the unique address of the resource itself

- **Inbound**: Can be directly accessed from the internet, the destination IP address is the unique address of the resource itself

Then we have resources residing in **Private Subnets**

- Each resource has only a private IP address that is used for local communication within the VPC

- **Outbound**: Can access the internet **indirectly**, thanks to a NAT Gateway, the source IP address is the public IP address of the NAT Gateway

- **Inbound**: Cannot be directly accessed from the internet, the NAT Gateway facilitates outbound traffic only, however any resource within the VPC can access it using its private IP address

A summary of routing in the diagram below

![](https://docs.aws.amazon.com/vpc/latest/userguide/images/public-nat-gateway-diagram.png)

### Route Tables

We shall create three route tables, 1 for the public subnets, and 1 for each private subnet, as every subnet will have a route to a different NAT Gateway

All the Route Tables should have the following properties defined

- **VPC ID**

  The ID of the VPC the route table is in, the VPC CIDR block will be the local route inside this route table, the local route is created by default at route table creation, so there's no need to define it explicitly

```yml
Resources:
  # ...
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  # ...
  PrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  # ...
  PrivateRouteTable2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
```

### Routes

#### Public Route

We shall have only one route for the public route table, which routes traffic to the internet gateway

Properties that need to be defined are

- **Route Table ID**

  The ID of the route table

- **Destination CIDR Block**

  The IPv4 CIDR block used for the destination match

- **Gateway ID**

  The ID of an internet gateway or virtual private gateway attached to your VPC

  If you create a route that references an Internet Gateway in the same template where you create the Internet Gateway, you must declare a dependency on the Internet Gateway attachment. The route table cannot use the Internet Gateway until it has successfully attached to the VPC. Add a `DependsOn` Attribute in the `AWS::EC2::Route` resource to explicitly declare a dependency on the `AWS::EC2::VPCGatewayAttachement` resource

```yml
Resources:
  # ...
  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayVPCAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
```

And then we shall associate the Public Route Table with the 2 Public Subnets

```yml
Resources:
  # ...
  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable
```

#### Private routes

A separate route for each NAT Gateway is added to the corresponding route table

Properties that need to be defined are

- **Route Table ID**

  The ID of the route table

- **Destination CIDR Block**

  The IPv4 CIDR block used for the destination match

- **NAT Gateway ID**

  The ID of a NAT gateway

```yml
Resources:
  # ...
  PrivateRoute1:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway1
  # ...
  PrivateRoute2:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway2
```

And then we shall associate the Private Route Tables with the Private Subnets

```yml
Resources:
  # ...
  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRouteTable1

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateRouteTable2
```

And by that we finalize the `Resources` section, now to the `Outputs`

## Outputs: VPC ID and Subnet IDs

We can add a parameter to our `Parameters` section to help us uniquely identify export names

```yml
Parameters:
  EnvironmentName:
    Description: The name of the deployment
    Type: String
    Default: My-HA-App
```

And then we can safely export all resource IDs that we will need later on

```yml
Outputs:
  VPC:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub "${EnvironmentName}-VPC"

  PublicSubnet1:
    Description: Public Subnet 1 ID
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub "${EnvironmentName}-Public-Subnet-1"

  PublicSubnet2:
    Description: Public Subnet 2 ID
    Value: !Ref PublicSubnet2
    Export:
      Name: !Sub "${EnvironmentName}-Public-Subnet-2"

  PrivateSubnet1:
    Description: Private Subnet 1 ID
    Value: !Ref PrivateSubnet1
    Export:
      Name: !Sub "${EnvironmentName}-Private-Subnet-1"

  PrivateSubnet2:
    Description: Private Subnet 2 ID
    Value: !Ref PrivateSubnet2
    Export:
      Name: !Sub "${EnvironmentName}-Private-Subnet-2"
```
