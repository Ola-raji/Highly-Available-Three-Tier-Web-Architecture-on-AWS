# Highly Available Three-Tier Web Architecture on AWS

![Cloud](https://img.shields.io/badge/Cloud-AWS-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Architecture](https://img.shields.io/badge/Architecture-Three--Tier%20HA-blue)

A hands-on cloud infrastructure project demonstrating a production-grade, highly available, reliable, scalable and fault-tolerant three-tier architecture on AWS, built from scratch using the AWS Management Console across two Availability Zones in us-east-1.

This project was built to develop and demonstrate practical understanding of core AWS services, network design, security layering, and failure recovery patterns. Every component was manually configured to reinforce how each piece fits together, and then intentionally broken to prove the high availability actually works.

---

## Objectives

- Design and deploy a custom VPC with public and private subnet segmentation across two AZs
- Build a secure compute layer using EC2 with no open SSH ports (SSM Session Manager only)
- Deploy an Application Load Balancer with health-check-driven traffic routing
- Configure an Auto Scaling Group for automated recovery and demand-based scaling
- Set up a managed MySQL database (RDS) with proper network isolation
- Instrument the full stack with CloudWatch dashboards, alarms, and SNS notifications
- Validate high availability through intentional failure injection and observation

---

## System Design Considerations

**High Availability**
Resources are distributed across two physically separate Availability Zones. The ALB, ASG, and RDS subnet group all span both AZs. No single component failure can take down the entire application — traffic reroutes automatically, instances self-replace via ASG, and the database has a standby path via Multi-AZ promotion. The minimum ASG capacity is set to 2 (not 1) to ensure at least one instance is always available in each AZ even during a scale-in event.

**Scalability**
The Auto Scaling Group responds to CPU demand without manual intervention. A target tracking policy maintains average fleet CPU at 50%, adding instances when demand rises and removing them during quiet periods. The ALB distributes load evenly across however many instances are running at any given time, so scaling in or out is completely transparent to users.

**Fault Tolerance**
Three failure classes were tested and validated against the architecture:
- *Hardware failure:* EC2 termination → ASG detects count below minimum and replaces instance automatically
- *Application failure:* Nginx process stopped → ALB detects unhealthy target within 20 seconds and reroutes traffic, even though the EC2 instance itself remains running
- *Capacity event:* CPU stress → ASG scales out beyond desired count, adds instance, then scales back in after cooldown

**Reliability**
Automated backups are enabled on RDS. CloudWatch alarms monitor CPU utilisation, healthy host count, and database storage across all tiers, with SNS email notifications on breach. The architecture recovers from all three failure scenarios above without any manual intervention.

**Security**
Layered security group chaining enforces least privilege at the network level — each tier only trusts the tier directly above it. Private subnets isolate application and database tiers from direct internet exposure. SSM Session Manager eliminates the open SSH attack surface entirely, and IMDSv2 prevents credential leakage via SSRF.

---

## Deployment Summary

All infrastructure was deployed manually via the AWS Management Console in us-east-1 (N. Virginia). The deployment followed this sequence:

1. Custom VPC and subnet layout across two AZs
2. Internet Gateway, NAT Gateway, and route table configuration
3. Security groups with chained source rules across all three tiers
4. IAM role for SSM Session Manager access
5. EC2 instances with IMDSv2-compatible User Data bootstrap script
6. Application Load Balancer with target group and health checks
7. Auto Scaling Group with launch template and target tracking policy
8. RDS MySQL instance in private DB subnet group
9. CloudWatch dashboard, alarms, and SNS notification topic

See the `/docs` folder for detailed notes on each phase, and `/architecture/DESIGN-DECISIONS.md` for the reasoning behind key configuration choices.

---

## Architecture Diagram

![Architecture Diagram](architecture/ha-architecture-diagram.png)

---

## Architecture Summary

The architecture follows a classic three-tier pattern — presentation, application, and data — deployed across two AWS Availability Zones (us-east-1a and us-east-1b) for fault tolerance.

**Networking layer**
A custom VPC (10.0.0.0/16) with four subnets: two public subnets hosting the ALB and NAT Gateway, and two private subnets hosting EC2 instances and RDS. An Internet Gateway handles inbound public traffic; a NAT Gateway provides outbound internet access for private resources without exposing them inbound. Each subnet tier has a dedicated route table — the public route table sends `0.0.0.0/0` to the IGW, and the private route table sends `0.0.0.0/0` through the NAT Gateway.

**Compute layer**
EC2 instances (Amazon Linux 2023, t3.micro) run Nginx and are managed by an Auto Scaling Group. The ASG maintains a minimum of 2 instances across both AZs, uses ELB health checks to detect application-level failures, and scales on a 50% average CPU target tracking policy up to a maximum of 4 instances. Instances are bootstrapped automatically at launch via a User Data script that installs Nginx and generates a custom page showing each instance's AZ and ID — making load balancer distribution visually observable.

**Traffic layer**
An internet-facing Application Load Balancer distributes HTTP traffic across the ASG fleet. Health checks run every 10 seconds with a 2-check failure threshold — an unhealthy instance is removed from rotation within 20 seconds, and re-added within 20 seconds of recovery, with no manual intervention.

**Data layer**
A MySQL 8.0 RDS instance (db.t3.micro) sits in a private DB subnet group spanning both AZs. It is not publicly accessible and only accepts connections from EC2 instances via security group chain on port 3306. The subnet group spanning both AZs means the instance can be promoted to Multi-AZ without any network reconfiguration.

**Security**
Security is enforced through three layers of security group chaining: the ALB accepts traffic from the internet, EC2 accepts traffic only from the ALB's security group, and RDS accepts traffic only from the EC2 security group on port 3306. No SSH port is open anywhere — all instance access uses SSM Session Manager, which provides browser-based terminal access without any open ports. IMDSv2 is enforced on all instances to prevent SSRF attacks against the metadata endpoint.

---

## AWS Services Used

| Service | Role in Architecture |
|---|---|
| Amazon VPC | Custom network with public/private subnet segmentation |
| EC2 | Application servers running Nginx (Amazon Linux 2023, t3.micro) |
| Auto Scaling Group | Automated fleet management, self-healing, and demand-based scaling |
| Application Load Balancer | Traffic distribution and health-check-based routing |
| Amazon RDS (MySQL 8.0) | Managed relational database in private subnet |
| AWS IAM | EC2 instance role for SSM access (least privilege) |
| SSM Session Manager | Secure instance access with no open SSH ports |
| Amazon CloudWatch | Metrics, dashboards, and alarms across all three tiers |
| Amazon SNS | Alarm notifications via email |
| NAT Gateway | Outbound internet access for private subnet resources |
| Internet Gateway | Inbound internet traffic entry point |

---

## Live Access
[ALB DNS](http://ha-app-load-balancer-121171499.us-east-1.elb.amazonaws.com)

Screenshots demonstrating the running architecture are available in the `/screenshots` directory, organised by component.

| Folder | Contents |
|---|---|
| `/screenshots/01-networking` | VPC, subnets, route tables, IGW, NAT Gateway |
| `/screenshots/02-compute` | EC2 instances, ASG activity, launch template |
| `/screenshots/03-load-balancer` | ALB config, target group, healthy targets |
| `/screenshots/04-database` | RDS instance, subnet group, connectivity |
| `/screenshots/05-monitoring` | CloudWatch dashboard, alarms, SNS topic |
| `/screenshots/06-resilience-testing` | Instance termination recovery, health check failure, scale-out event |
