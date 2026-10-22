# AWS Highly Available Web Architecture Using Application Load Balancer

## 1. Project Overview

This project demonstrates the deployment of a highly available web application architecture on Amazon Web Services (AWS). It uses a custom Virtual Private Cloud (VPC), public and private subnets across three Availability Zones, NAT Gateways, private EC2 web servers, and an internet-facing Application Load Balancer.

Nginx is installed on three Ubuntu EC2 instances. Each instance serves a custom HTML page identifying the backend server. The Application Load Balancer distributes incoming HTTP requests to healthy targets in the target group.

**Project Type:** Cloud Infrastructure and Networking  
**Cloud Provider:** Amazon Web Services (AWS)  
**AWS Region:** Asia Pacific (Singapore) — `ap-southeast-1`

## 2. Project Objectives

- Create a custom VPC with an organized IP address range.
- Configure public and private subnets across three Availability Zones.
- Configure Internet Gateway and NAT Gateways for network connectivity.
- Deploy three EC2 web servers in private subnets.
- Configure Nginx to serve custom web pages.
- Configure an Application Load Balancer to distribute HTTP traffic.
- Configure target group health checks.
- Apply security group rules and network access controls.
- Test application accessibility and backend responses.

## 3. AWS Services Used

| AWS Service | Purpose |
|---|---|
| Amazon VPC | Isolated virtual network |
| Public and Private Subnets | Network segmentation |
| Internet Gateway | Internet connectivity for public resources |
| NAT Gateways | Outbound internet access for private resources |
| Route Tables | Control network traffic routing |
| Amazon EC2 | Hosts the web servers |
| Ubuntu Linux | Operating system for web servers |
| Nginx | Web server software |
| Application Load Balancer | Distributes incoming HTTP requests |
| Target Group | Registers web servers and monitors their health |
| Security Groups | Controls instance and load balancer traffic |
| Network ACL | Subnet-level network traffic filtering |
| AWS Systems Manager | Supports instance management through Session Manager |

## 4. VPC and Network Configuration

A custom VPC named `project-vpc` was created with the CIDR block `10.0.0.0/16`.

**VPC ID:** `vpc-0cfeae02ab1f9b2c8`

### Public Subnets

Public subnets host internet-facing resources and provide network connectivity to the Application Load Balancer.

| Subnet | Availability Zone | CIDR Block |
|---|---|---|
| project-subnet-public1 | ap-southeast-1a | 10.0.0.0/20 |
| project-subnet-public2 | ap-southeast-1b | 10.0.16.0/20 |
| project-subnet-public3 | ap-southeast-1c | 10.0.32.0/20 |

### Private Subnets

Private subnets host the EC2 web servers without requiring direct public internet access to those instances.

| Subnet | Availability Zone | CIDR Block |
|---|---|---|
| project-subnet-private1 | ap-southeast-1a | 10.0.128.0/20 |
| project-subnet-private2 | ap-southeast-1b | 10.0.144.0/20 |
| project-subnet-private3 | ap-southeast-1c | 10.0.160.0/20 |

The subnet names identify the Availability Zones in which the resources are intended to reside.

## 5. Network Routing

### Internet Gateway

An Internet Gateway named `project-igw` is attached to the VPC.

The public route table, `project-rtb-public`, includes these routes:

| Destination | Target |
|---|---|
| 10.0.0.0/16 | Local VPC routing |
| 0.0.0.0/0 | Internet Gateway |

The public route table is associated with the three public subnets.

### NAT Gateways

Three NAT Gateways were created for the architecture, with one intended for each Availability Zone:

- `project-nat-public1-ap-southeast-1a`
- `project-nat-public2-ap-southeast-1b`
- `project-nat-public3-ap-southeast-1c`

NAT Gateways allow private-subnet resources to initiate supported outbound connections to the internet without accepting unsolicited inbound internet connections through the NAT Gateway.

Private route tables should route internet-bound traffic (`0.0.0.0/0`) through the corresponding NAT Gateway. Verify their active routes and subnet associations in the AWS Console before treating this part as fully validated.

**Cost note:** NAT Gateways incur charges while provisioned, as well as data-processing charges. Review and delete unneeded resources when the project is not in use.

## 6. EC2 Web Servers

Three EC2 instances were configured in private subnets:

- `project-web-1`
- `project-web-2`
- `project-web-3`

The instances run Ubuntu Linux and use Nginx to serve web content over HTTP on port 80.

Each server displays a different response:

- `Hello from project-web-1`
- `Hello from project-web-2`
- `Hello from project-web-3`

These distinct responses make it easier to identify which backend server handled a request.

## 7. Application Load Balancer

An internet-facing Application Load Balancer named `project-alb` was created across the three public subnets.

**DNS name:**

`project-alb-782290905.ap-southeast-1.elb.amazonaws.com`

**Listener:** HTTP on port 80  
**Target group:** `project-web-tg`

The listener forwards incoming HTTP requests to the target group, which contains the three EC2 web servers.

### Target Group Configuration

| Setting | Value |
|---|---|
| Target group name | project-web-tg |
| Target type | Instance |
| Protocol | HTTP |
| Port | 80 |
| Protocol version | HTTP/1.1 |
| Health check path | `/` |
| Success code | 200 |

All three targets were reported as healthy during testing.

## 8. Security Configuration

### Application Load Balancer Security Group

Security group: `project-alb-sg`

- Allow inbound HTTP traffic on port 80 from the internet.
- Retain only the outbound rules required for communication with the targets.

### Web Server Security Group

Security group: `project-web-sg`

Recommended configuration:

- Allow inbound HTTP traffic on port 80 only from `project-alb-sg`.
- Avoid exposing SSH port 22 to the entire internet.
- Use AWS Systems Manager Session Manager for administration where configured.

The web server security group should reference the load balancer security group as the source for HTTP traffic. Verify that these recommended changes have been applied in the AWS Console.

### Network ACL

The VPC's main Network ACL was configured with allow rules for inbound and outbound traffic and default deny rules for unmatched traffic.

Network ACLs are stateless, so return traffic must be permitted by the applicable rules. The current permissive rules are suitable for basic connectivity testing but should be reviewed and restricted before production use.

## 9. Architecture Diagram

```text
                        Internet
                           |
                           v
                    Internet Gateway
                           |
                           v
                Application Load Balancer
                      HTTP : 80
                           |
                Target Group: project-web-tg
                           |
           +---------------+---------------+
           |               |               |
           v               v               v
       EC2 Web 1       EC2 Web 2       EC2 Web 3
       Nginx : 80      Nginx : 80      Nginx : 80
           |               |               |
     Private Subnet  Private Subnet  Private Subnet
        AZ: 1a          AZ: 1b          AZ: 1c

        Private instances can use NAT Gateways
        for outbound internet access when routes
        and security rules permit it.
```

## 10. Testing and Validation

### Test 1: Load Balancer Connectivity

The load balancer DNS name was tested using Windows Command Prompt:

```cmd
curl.exe -v --connect-timeout 10 http://project-alb-782290905.ap-southeast-1.elb.amazonaws.com/
```

The request returned:

```text
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
```

This confirmed successful HTTP connectivity through the load balancer to a web server.

### Test 2: Backend Identification

After replacing the default Nginx welcome page with custom HTML pages, each server displayed its own identifying message.

A repeated-request test can be run in Windows Command Prompt:

```cmd
for /L %i in (1,1,10) do @curl.exe -s http://project-alb-782290905.ap-southeast-1.elb.amazonaws.com/ | findstr /i "Hello from"
```

The output can help demonstrate responses from different backend servers. Repeated requests do not guarantee that every server will appear in every test.

### Test 3: Target Health

The target group health status was checked in the AWS Console. All three registered targets were reported as healthy.

## 11. Challenges and Troubleshooting

### Initial Load Balancer Timeout

Initially, the load balancer website did not respond and the browser displayed a timeout error. Later, testing with `curl.exe` successfully returned HTTP 200.

This demonstrated the importance of validating connectivity using multiple methods and checking listener configuration, target health, routes, and security group rules when troubleshooting.

### Default Nginx Page

Initially, Nginx served its default welcome page. The web content was subsequently replaced with custom HTML pages to identify each backend instance.

### Health Checks

The target group health check uses the `/` path and expects an HTTP 200 response. The web server must be running, the page must be accessible, and network rules must permit the health-check traffic.

## 12. High Availability and Scalability

The architecture distributes web servers across three Availability Zones and uses an Application Load Balancer to route traffic to healthy registered targets.

This design can improve availability and provide a foundation for horizontal scaling. However, three instances alone do not guarantee complete fault tolerance. Production readiness also depends on correct routing, security controls, application design, monitoring, capacity planning, and recovery procedures.

## 13. Future Improvements

Potential enhancements include:

- Use an EC2 Auto Scaling Group to maintain and scale the desired number of instances.
- Manage the infrastructure using Terraform.
- Configure HTTPS using AWS Certificate Manager.
- Use AWS Systems Manager instead of publicly exposed SSH access.
- Add monitoring and alarms using Amazon CloudWatch.
- Restrict security group and Network ACL rules according to least-privilege principles.
- Automate Nginx installation and deployment.
- Add a custom domain using Amazon Route 53.

## 14. Cost and Cleanup

This project uses AWS resources that may incur charges, including EC2 instances, NAT Gateways, public IPv4 addresses, and load balancing.

Before finishing the project, review all resources in the AWS Console. When the environment is no longer needed, safely delete resources in the correct dependency order, including the load balancer, target group, NAT Gateways and their Elastic IP allocations, EC2 instances, and other networking resources. Do not delete resources that are still needed.

## 15. Conclusion

This project demonstrates practical cloud infrastructure skills, including VPC networking, subnet design, route tables, Internet and NAT Gateways, EC2 administration, Nginx configuration, security groups, load balancing, and health-check troubleshooting.

It provides a foundation for further learning in cloud engineering, infrastructure automation, and DevOps.

Author: Kavya Mane
Repository: AWS Highly Available Web Architecture
