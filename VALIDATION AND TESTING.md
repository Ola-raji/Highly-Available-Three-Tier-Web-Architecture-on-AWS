## Validation and Testing ⚙️

Three intentional failure tests were conducted to validate HA behaviour.

---
**Test 1 - EC2 Instance Termination**:
One ASG-managed instance was terminated via the console. The ALB detected the failure within 20 seconds and rerouted all traffic to the surviving instance. The ASG automatically launched a replacement, which passed health checks and rejoined the target group within approximately 4 minutes. No manual intervention was required at any point.

---

**Test 2 - Application-Layer Health Check Failure**:
Nginx was stopped on a running instance via SSM Session Manager. The ALB detected the unhealthy response within 20 seconds and removed the instance from rotation while the EC2 instance itself remained running. The CloudWatch alarm fired and an SNS email notification was received. Nginx was restarted and the instance recovered to healthy within 20 seconds.

---

**Test 3 - CPU Stress and Auto Scaling**:
The `stress` tool was run simultaneously on both instances to push fleet average CPU above the 50% target. The ASG launched a third instance after two consecutive 5-minute periods above threshold. After the stress period ended, the ASG scaled back in to the desired count of 2 during the cooldown window.
