
# Requirements

## Server specs

- You'll need to create a **Launch Configuration** for your application servers in order to deploy **four** servers, **two located in each of your private subnets**. The launch configuration will be used by an auto-scaling group

- You'll need **two vCPUs** and at least **4 GB of RAM**. The Operating System to be used is **Ubuntu**. So, choose an Instance size and Machine Image (AMI) that best fits this spec

- Be sure to allocate at least **10 GB** of disk space so that you don't run into issues

## Routing and Security Groups

### Web Servers

- **Inbound**
    - Udagram communicates on the default **HTTP Port: 80**, so your servers will need this **inbound port open** since you will use it with the **Load Balancer** and the **Load Balancer Health Check**

- **Outbound**
    - The servers will need **unrestricted** internet access to be able to download and update their software

### Load balancer

- **Inbound**
    - The load balancer should allow all public traffic (0.0.0.0/0) on port **80 inbound**, which is the default HTTP port. Outbound, it will only be using port 80 to reach the internal servers

- **Outbound**
    - The application needs to be deployed into **private subnets** with a **Load Balancer located in a public subnet**


## IAM Access and Roles

- Since you will be downloading the application archive from an S3 Bucket, you'll need to create an **IAM Role** that allows your instances to use the **S3 Service**


## Extra

- One of the output exports of the CloudFormation script should be the **public URL** of the Load Balancer. Bonus points if you add http:// in front of the load balancer DNS Name in the output, for convenience

## Prerequisites

- An S3 Bucket hosted on the account

- A simple index.html file uploaded to the bucket, you can find a [sample](../assets/index.html) in the assets` folder
