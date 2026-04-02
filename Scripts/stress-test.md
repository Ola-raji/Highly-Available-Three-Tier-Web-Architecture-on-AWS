## stress-test.sh — CPU Stress Test for ASG Scale-Out

```bash
#!/bin/bash
# CPU Stress Test Script
# Runs on EC2 instances via SSM Session Manager
# Triggers Auto Scaling scale-out by pushing CPU above 50% target
# Must be run on ALL instances simultaneously to exceed fleet average threshold
#
# Usage: Run as root (sudo su - first)
# Duration: 300 seconds (5 minutes) — auto-stops

echo "Installing stress tool..."
yum install -y stress

echo "Starting CPU stress — 2 workers for 300 seconds"
echo "Monitor ASG Activity tab and CloudWatch CPUUtilization graph"
echo "Scale-out should trigger within 5-10 minutes if run on all instances"

stress --cpu 2 --timeout 300

echo "Stress test complete. Monitor for scale-in over next 10-15 minutes."
```
