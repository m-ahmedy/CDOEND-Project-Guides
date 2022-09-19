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

Check this valuable reource on subnetting and PV

| Item             | CIDR        | Parameter Name     |
| ---------------- | ----------- | ------------------ |
| VPC              | 10.0.0.0/16 | VPCCIDR            |
| Private Subnet 1 | 10.0.0.0/24 | PrivateSubnet1CIDR |
| Public Subnet 1  | 10.0.1.0/24 | PublicSubnet1CIDR  |
| Private Subnet 2 | 10.0.2.0/24 | PrivateSubnet2CIDR |
| Public Subnet 2  | 10.0.3.0/24 | PublicSubnet2CIDR  |

## Resources

### VPC

The VPC Resource should have the following properties defined

- **CIDR Block**

  The IPv4 network range for the VPC, in CIDR notation

- **DNS Support**

  Indicates whether the DNS resolution is supported for the VPC. If enabled, queries to the Amazon provided DNS server

- **DNS Hostnames**

  Indicates whether the instances launched in the VPC get DNS hostnames. If enabled, instances in the VPC get DNS hostnames

```yml
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
InternetGateway:
  Type: AWS::EC2::InternetGateway
```

#### VPC Gateway Attachment

After creating the Internet gateway, you then attach it to our VPC

- **VPC ID**

  The ID of the VPC

- **Internet Gateway ID**

  The ID of the internet gateway

```yml
InternetGatewayVPCAttachment:
  Type: AWS::EC2::VPCGatewayAttachment
  Properties:
    InternetGatewayId: !Ref InternetGateway
    VpcId: !Ref VPC
```
