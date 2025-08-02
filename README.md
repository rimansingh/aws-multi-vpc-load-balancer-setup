# AWS Global Accelerator & Multi-Region Networking

This document outlines the steps to design and implement a highly available and low-latency global application architecture using AWS services. By leveraging AWS Global Accelerator, we can intelligently route user traffic to the nearest healthy regional endpoint, thereby improving uptime and performance.

---

### Architecture Diagram

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
        ALB1 --> Listener1[Listener (HTTP/HTTPS)]
        Listener1 --> TG1[Target Group]
        TG1 --> EC2_1_1[EC2 - Web Server 1 (AZ-a)]
        TG1 --> EC2_1_2[EC2 - Web Server 2 (AZ-b)]
    end

    subgraph "eu-central-1 (Frankfurt) Region"
        EndpointGroup2 --> ALB2[Application Load Balancer]
        ALB2 --> Listener2[Listener (HTTP/HTTPS)]
        Listener2 --> TG2[Target Group]
        TG2 --> EC2_2_1[EC2 - Web Server 1 (AZ-a)]
        TG2 --> EC2_2_2[EC2 - Web Server 2 (AZ-b)]
    end

    subgraph "VPC - Mumbai (21.0.0.0/16)"
        VPC1[VPC]
        Subnet1A[Subnet-A (21.0.1.0/24 - AZ-a)]
        Subnet1B[Subnet-B (21.0.2.0/24 - AZ-b)]
        IGW1[Internet Gateway]
        RT1[Route Table]

        VPC1 --> Subnet1A
        VPC1 --> Subnet1B
        EC2_1_1 --> Subnet1A
        EC2_1_2 --> Subnet1B
        ALB1 --> Subnet1A
        ALB1 --> Subnet1B
        VPC1 --> IGW1
        Subnet1A --> RT1
        Subnet1B --> RT1
        RT1 --> IGW1
    end

    subgraph "VPC - Frankfurt (31.0.0.0/16)"
        VPC2[VPC]
        Subnet2A[Subnet-A (31.0.1.0/24 - AZ-a)]
        Subnet2B[Subnet-B (31.0.2.0/24 - AZ-b)]
        IGW2[Internet Gateway]
        RT2[Route Table]

        VPC2 --> Subnet2A
        VPC2 --> Subnet2B
        EC2_2_1 --> Subnet2A
        EC2_2_2 --> Subnet2B
        ALB2 --> Subnet2A
        ALB2 --> Subnet2B
        VPC2 --> IGW2
        Subnet2A --> RT2
        Subnet2B --> RT2
        RT2 --> IGW2
    end
```

---
## Outcomes

*   **Improved Global Application Uptime:** By distributing traffic across multiple regions, the application remains available even if one region experiences an outage.
*   **Reduced Latency:** User traffic is intelligently routed to the AWS edge location closest to them and then over the AWS global network to the nearest healthy application endpoint, resulting in lower latency and better performance.

---

## Step-by-Step Implementation Guide

Here is a detailed breakdown of the steps required to configure this architecture:

### 1. VPC and Subnet Creation

First, we will create two Virtual Private Clouds (VPCs), one in the Mumbai (`ap-south-1`) region and another in the Frankfurt (`eu-central-1`) region.

**In the Mumbai (`ap-south-1`) Region:**

1.  **Create the VPC:**
    *   Navigate to the VPC dashboard in the AWS Management Console.
    *   Click on "Create VPC".
    *   **Name tag:** `VPC-Mumbai`
    *   **IPv4 CIDR block:** `21.0.0.0/16`
    *   Leave other settings as default and create the VPC.
2.  **Create a Subnet:**
    *   In the VPC dashboard, go to "Subnets" and click "Create subnet".
    *   **VPC ID:** Select `VPC-Mumbai`.
    *   **Subnet name:** `Public-Subnet-Mumbai`
    *   **Availability Zone:** Choose an AZ in the `ap-south-1` region.
    *   **IPv4 CIDR block:** `21.0.1.0/24`
3.  **Create and Attach an Internet Gateway:**
    *   In the VPC dashboard, go to "Internet Gateways" and click "Create internet gateway".
    *   **Name tag:** `IGW-Mumbai`
    *   After creation, attach it to `VPC-Mumbai`.
4.  **Configure the Route Table:**
    *   Go to "Route Tables" and select the main route table associated with `VPC-Mumbai`.
    *   Add a route:
        *   **Destination:** `0.0.0.0/0`
        *   **Target:** Select the `IGW-Mumbai` you created.

**In the Frankfurt (`eu-central-1`) Region:**

Repeat the same steps as above, but with the following configurations:

1.  **Create the VPC:**
    *   **Name tag:** `VPC-Frankfurt`
    *   **IPv4 CIDR block:** `31.0.0.0/16`
2.  **Create a Subnet:**
    *   **VPC ID:** Select `VPC-Frankfurt`.
    *   **Subnet name:** `Public-Subnet-Frankfurt`
    *   **Availability Zone:** Choose an AZ in the `eu-central-1` region.
    *   **IPv4 CIDR block:** `31.0.1.0/24`
3.  **Create and Attach an Internet Gateway:**
    *   **Name tag:** `IGW-Frankfurt`
    *   Attach it to `VPC-Frankfurt`.
4.  **Configure the Route Table:**
    *   Select the main route table for `VPC-Frankfurt`.
    *   Add a route:
        *   **Destination:** `0.0.0.0/0`
        *   **Target:** Select the `IGW-Frankfurt`.

### 2. EC2 Instance and Apache Web Server Setup

Now, launch and configure EC2 instances in each public subnet to act as web servers.

**In Both Regions (Mumbai and Frankfurt):**

1.  **Launch EC2 Instances:**
    *   Navigate to the EC2 dashboard.
    *   Click "Launch instances".
    *   Choose an Amazon Machine Image (AMI), such as Amazon Linux 2.
    *   Select an instance type (e.g., `t2.micro` for testing).
    *   **Network:** Select the respective VPC (`VPC-Mumbai` or `VPC-Frankfurt`).
    *   **Subnet:** Select the public subnet (`Public-Subnet-Mumbai` or `Public-Subnet-Frankfurt`).
    *   **Auto-assign Public IP:** Enable this.
2.  **Configure Security Groups:**
    *   Create a new security group for each region with the following inbound rules:
        *   **Type:** `SSH` | **Protocol:** `TCP` | **Port Range:** `22` | **Source:** Your IP address.
        *   **Type:** `HTTP` | **Protocol:** `TCP` | **Port Range:** `80` | **Source:** `0.0.0.0/0`.
3.  **Install Apache Web Server:**
    *   Connect to your EC2 instances via SSH.
    *   Run the following commands to install and start the Apache web server:
        ```bash
        sudo yum update -y
        sudo yum install -y httpd
        sudo systemctl start httpd
        sudo systemctl enable httpd
        ```
    *   Create a simple `index.html` file to identify the server's region:
        *   **For Mumbai:** `echo "<h1>Welcome from Mumbai Region</h1>" | sudo tee /var/www/html/index.html`
        *   **For Frankfurt:** `echo "<h1>Welcome from Frankfurt Region</h1>" | sudo tee /var/www/html/index.html`

### 3. Application Load Balancer (ALB) and Target Group Configuration

Next, set up an Application Load Balancer in each region to distribute traffic to the EC2 instances.

**In Both Regions (Mumbai and Frankfurt):**

1.  **Create a Target Group:**
    *   In the EC2 dashboard, go to "Target Groups" under "Load Balancing".
    *   Click "Create target group".
    *   **Target type:** `Instances`
    *   **Target group name:** `TG-Mumbai` or `TG-Frankfurt`
    *   **Protocol:** `HTTP` | **Port:** `80`
    *   **VPC:** Select the respective VPC.
    *   Register your EC2 instances in that region as targets.
2.  **Create an Application Load Balancer:**
    *   In the EC2 dashboard, go to "Load Balancers" and click "Create Load Balancer".
    *   **Type:** `Application Load Balancer`
    *   **Name:** `ALB-Mumbai` or `ALB-Frankfurt`
    *   **Scheme:** `Internet-facing`
    *   **VPC:** Select the respective VPC.
    *   **Mappings:** Select the public subnet in the chosen Availability Zone.
    *   **Security Groups:** Create a new security group that allows inbound HTTP traffic (port 80) from `0.0.0.0/0`.
    *   **Listeners and routing:** Forward traffic to the target group you created earlier.

### 4. AWS Global Accelerator Configuration

Finally, set up AWS Global Accelerator to direct traffic to the regional Application Load Balancers.

1.  **Create an Accelerator:**
    *   Navigate to the AWS Global Accelerator console.
    *   Click "Create accelerator".
    *   **Accelerator name:** Provide a descriptive name.
    *   Global Accelerator will provide you with two static IP addresses.
2.  **Add a Listener:**
    *   **Port range:** `80`
    *   **Protocol:** `TCP`
3.  **Add Endpoint Groups:**
    *   **Endpoint group region:** `ap-south-1` (Mumbai)
    *   **Add endpoint:**
        *   **Endpoint type:** `Application Load Balancer`
        *   **Endpoint:** Select the `ALB-Mumbai` you created.
    *   Repeat this process to add another endpoint group for the Frankfurt region:
        *   **Endpoint group region:** `eu-central-1` (Frankfurt)
        *   **Add endpoint:**
            *   **Endpoint type:** `Application Load Balancer`
            *   **Endpoint:** Select the `ALB-Frankfurt`.
