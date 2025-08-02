# AWS Global Accelerator & Multi-Region Networking

This document outlines the steps to design and implement a highly available and low-latency global application architecture using AWS services. By leveraging AWS Global Accelerator, we can intelligently route user traffic to the nearest healthy regional endpoint, thereby improving uptime and performance.

---

## Architecture Diagram

```mermaid
graph TD
subgraph "User Traffic"
direction LR
User -->|Internet| GlobalAccelerator
end

subgraph "AWS Global Network"
    GlobalAccelerator[AWS Global Accelerator] -->|Intelligent Routing| EndpointGroup1[Endpoint Group - Mumbai]
    GlobalAccelerator -->|Intelligent Routing| EndpointGroup2[Endpoint Group - Frankfurt]
end

subgraph "ap-south-1 (Mumbai) Region"
    EndpointGroup1 --> ALB1[Application Load Balancer]
    ALB1 --> TG1[Target Group]
    TG1 --> EC2_1_1[EC2 - Web Server 1]
    TG1 --> EC2_1_2[EC2 - Web Server 2]
end

subgraph "eu-central-1 (Frankfurt) Region"
    EndpointGroup2 --> ALB2[Application Load Balancer]
    ALB2 --> TG2[Target Group]
    TG2 --> EC2_2_1[EC2 - Web Server 1]
    TG2 --> EC2_2_2[EC2 - Web Server 2]
end

subgraph "VPC - Mumbai (21.0.0.0/16)"
    VPC1[VPC]
    Subnet1[Subnet - 21.0.1.0/24]
    IGW1[Internet Gateway]
    RT1[Route Table]

    VPC1 -- contains --> Subnet1
    EC2_1_1 -- resides in --> Subnet1
    EC2_1_2 -- resides in --> Subnet1
    ALB1 -- spans --> Subnet1
    VPC1 -- attached to --> IGW1
    Subnet1 -- associated with --> RT1
    RT1 -- routes to --> IGW1
end

subgraph "VPC - Frankfurt (31.0.0.0/16)"
    VPC2[VPC]
    Subnet2[Subnet - 31.0.1.0/24]
    IGW2[Internet Gateway]
    RT2[Route Table]

    VPC2 -- contains --> Subnet2
    EC2_2_1 -- resides in --> Subnet2
    EC2_2_2 -- resides in --> Subnet2
    ALB2 -- spans --> Subnet2
    VPC2 -- attached to --> IGW2
    Subnet2 -- associated with --> RT2
    RT2 -- routes to --> IGW2
end