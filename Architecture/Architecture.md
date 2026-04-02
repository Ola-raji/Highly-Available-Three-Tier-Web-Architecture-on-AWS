# Architecture Overview

A highly available three-tier web application deployed across two Availability Zones (**us-east-1a** and **us-east-1b**) inside a custom VPC (`10.0.0.0/16`). Each tier is isolated in its own subnet layer with access controlled entirely through security group chaining.

---

## Network

The VPC contains four subnets — one public and one private per AZ. Public subnets route outbound traffic to an **Internet Gateway**. Private subnets route outbound traffic through a **NAT Gateway** (with Elastic IP) hosted in the public subnet, keeping EC2 and RDS instances unreachable from the internet.

| Subnet | CIDR | AZ |
|---|---|---|
| ha-public-subnet-1a | 10.0.1.0/24 | us-east-1a |
| ha-public-subnet-1b | 10.0.2.0/24 | us-east-1b |
| ha-private-subnet-1a | 10.0.3.0/24 | us-east-1a |
| ha-private-subnet-1b | 10.0.4.0/24 | us-east-1b |

---

## Compute

An **Application Load Balancer** (`ha-app-load-balancer`) sits in the public subnets and distributes HTTP traffic across EC2 instances registered in the target group `ha-web-target-group`. Instances run **Nginx on Amazon Linux 2023** (t3.micro) and serve a dynamic page showing their instance ID and Availability Zone.

An **Auto Scaling Group** (`ha-web-asg`) spans both private subnets with a minimum of 2 and maximum of 4 instances. A target tracking policy scales out at 50% average CPU. ELB health checks are enabled so the ASG replaces unhealthy instances automatically.

---

## Database

**RDS MySQL 8.0** (`ha-project-db`, db.t3.micro) runs in a dedicated DB subnet group spanning both private subnets. Only EC2 instances are permitted to reach port 3306 via the `ha-rds-sg` security group. Multi-AZ is disabled for cost (free tier) but the subnet group is already multi-AZ ready for a one-click upgrade.

---

## Security

Traffic flows through a strict three-layer chain with no SSH open anywhere:

```
Internet → ha-alb-sg (port 80) → ha-ec2-sg (port 80) → ha-rds-sg (port 3306)
```

EC2 instances use **SSM Session Manager** for shell access via the `ha-ec2-ssm-role` IAM role. IMDSv2 (token-based metadata) is enforced on all instances.

---

## Monitoring

CloudWatch alarms notify via SNS email (`ha-project-alerts`) on: high ASG CPU, ALB unhealthy target count, EC2 status check failures, and low RDS storage. A CloudWatch dashboard (`ha-project-dashboard`) provides a single-pane view of all key metrics.
