# Highly Available Three-Tier Web Architecture on AWS 

![Cloud](https://img.shields.io/badge/Cloud-AWS-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Architecture](https://img.shields.io/badge/Architecture-Three--Tier%20HA-blue)

A hands-on cloud infrastructure project demonstrating a production-grade, highly available, reliable, scalable and fault-tolerant three-tier architecture on AWS, built from scratch using the AWS Management Console across two Availability Zones in us-east-1.

This project demonstrates practical in-depth understanding of AWS services, network design, security layering, and failure recovery patterns. Every component was manually configured to reinforce how each piece fits together, and then intentionally broken to prove the high availability actually works.

---

### Objectives

- Design and deploy a custom VPC with public and private subnet segmentation across two AZs
- Build a secure compute layer using EC2 with no open SSH ports (SSM Session Manager only)
- Deploy an Application Load Balancer with health-check-driven traffic routing
- Configure an Auto Scaling Group for automated recovery and demand-based scaling
- Set up a managed MySQL database (RDS) with proper network isolation
- Instrument the full stack with CloudWatch dashboards, alarms, and SNS notifications
- Validate high availability through intentional failure injection and observation

---

### Architecture Diagram
<img alt="HA Three-Tier AWS Architecture" src="https://github.com/user-attachments/assets/45e115e7-beff-4f0d-9814-84b6ff25d94b" width="510" />

---

### System Design Considerations

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

### Deployment Summary

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

---

### AWS Services Used

|Service|Role in Architecture|
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

### Live Access
[Load Balancer Preview](http://ha-app-load-balancer-121171499.us-east-1.elb.amazonaws.com)

---
### Project Limitations
- This project was built within AWS Free Tier constraints and is scoped for a lab environment rather than a production deployment. As a result, a number of deliberate trade-offs had to be made. 

- Only one NAT Gateway is deployed in us-east-1a as a result of resource constraints, a production setup would add a second in us-east-1b to eliminate the single point of failure for private subnet egress. 

- RDS runs in Single-AZ mode with 1-day backup retention; Multi-AZ is disabled to stay within free tier, though the DB subnet group already spans both AZs and can be upgraded with a single configuration change.

- The ALB serves HTTP only. Without a custom domain, ACM cannot issue a certificate, so HTTPS termination was not possible. Database credentials are managed manually rather than through AWS Secrets Manager, and application logs are not shipped to CloudWatch Logs, both acceptable for a lab environment but gaps that would need to be addressed before any production use.
