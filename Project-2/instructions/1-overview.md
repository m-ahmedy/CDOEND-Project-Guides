
# Part 1 - Overview

## What Is High Availability?

Highly available systems are **reliable** in the sense that they continue operating even when critical components fail. They are also **resilient**, meaning that they are able to simply handle failure without service disruption or data loss, and seamlessly recover from such failure.

### Highly Available Compute on AWS

Amazon EC2 and other services that let you provision computing resources, provide high availability features such as **load balancing**, **auto-scaling** and provisioning across Amazon **Availability Zones (AZ)**, representing isolated parts of an Amazon data center

If you are running instances on Amazon EC2, Amazon provides several built-in capabilities to achieve high availability:

- **Elastic Load Balancing** - you can launch several EC2 instances and distribute traffic between them

- **Availability Zones** - you can place instances in different AZs

- **Auto Scaling** - use auto-scaling to detect when loads increase, and then dynamically add more instances

These capabilities are illustrated in the diagram below. The Elastic Load Balancer distributes traffic between two or more EC2 instances, each of which can potentially be deployed in a separate subnet that resides in a separate Amazon Availability Zone. These instances can be part of an Auto-Scaling Group, with additional instances launched on demand

### Networking needs for HA Compute

Our Compute power (EC2 Auto scaling instances and Load Balancer) will require a network that spans multiple AZs

At least **one private subnet** and **one public subnet** for **two** (or more) availability zones
