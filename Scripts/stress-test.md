# stress-test.sh — CPU Stress Test for ASG Scale-Out

Installs and runs the `stress` tool on an EC2 instance to artificially push CPU utilisation to 100%. When run simultaneously on all instances in the fleet, the average CPU across the Auto Scaling Group exceeds the 50% target tracking threshold — triggering a scale-out event and demonstrating that the ASG scaling policy works as configured.

**Where to run:** On each ASG-managed EC2 instance via SSM Session Manager, as root.  
**When to run:** After the ASG and CloudWatch alarms are confirmed working. Run on both instances at the same time.

---

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

---

## Notes

**Why run on both instances simultaneously?**  
The ASG target tracking policy watches *average* fleet CPU, not individual instance CPU. If only one instance is stressed:

```
Instance 1a:  100% CPU
Instance 1b:    0% CPU
Fleet average:  50%  — right on the threshold, may not trigger
```

With both instances stressed:

```
Instance 1a:  100% CPU
Instance 1b:  100% CPU
Fleet average: 100% — well above threshold, scale-out triggers reliably
```

**Why `--cpu 2`?**  
A `t3.micro` has 2 vCPUs. Setting `--cpu 2` spawns two worker processes, each maxing out one core — pegging both vCPUs at 100%.

**Why `--timeout 300`?**  
Automatically stops after 5 minutes. This prevents the test running indefinitely if the terminal session is closed, and gives enough sustained load for CloudWatch to record two consecutive 5-minute data points above the threshold — which is what triggers the alarm and scale-out.

**What to observe:**  
CloudWatch `CPUUtilization` graph spikes to near 100% → `ha-asg-high-cpu` alarm enters ALARM state → ASG Activity tab shows a new launch event → a third instance appears in EC2 with state Pending then Running → Target Group registers a third healthy target. After the stress ends, the fleet CPU drops back to near 0% and the ASG scales back in to the desired count of 2 over the following 10–15 minutes.
