# generate-traffic.sh — ALB Traffic Simulation

Sends a configurable number of HTTP requests to the ALB DNS endpoint to generate visible traffic metrics in CloudWatch. Used to populate the RequestCount graph on the CloudWatch dashboard and demonstrate the ALB distributing requests across both Availability Zones.

**Where to run:** Local machine terminal, AWS CloudShell, or any host outside the VPC. Running from outside the VPC ensures traffic follows the real path — internet → ALB → EC2 — rather than an internal VPC shortcut.  
**When to run:** After the ALB and target group are confirmed healthy, before or during CloudWatch dashboard review.

---

```bash
#!/bin/bash
# ALB Traffic Simulation Script
# Sends N requests to the ALB to generate CloudWatch metrics
# Usage: ./generate-traffic.sh <ALB_DNS> <REQUEST_COUNT>
# Example: ./generate-traffic.sh my-alb.us-east-1.elb.amazonaws.com 100

ALB_DNS=${1:-"YOUR-ALB-DNS-HERE"}
COUNT=${2:-50}

echo "Sending $COUNT requests to http://$ALB_DNS"
echo "Watch CloudWatch RequestCount metric for results..."

for i in $(seq 1 $COUNT); do
  curl -s "http://$ALB_DNS" > /dev/null
  echo -ne "Request $i/$COUNT\r"
done

echo ""
echo "Done. Allow 5 minutes for metrics to appear in CloudWatch."
```

---

## Notes

**Why run from outside the VPC?**  
Traffic originating from inside the VPC (e.g. from an SSM session on EC2) takes a different network path and may bypass the ALB entirely. Running from CloudShell or a local machine ensures the full request lifecycle is exercised — and that the metrics recorded in CloudWatch reflect real external traffic.

**Why `> /dev/null`?**  
Suppresses the HTML response output in the terminal. The request still completes fully and is counted by the ALB — the response is just discarded locally since we're only interested in the CloudWatch metrics, not the content.

**What to observe in CloudWatch:**  
After running, wait 5 minutes then check the `RequestCount` metric on `ha-app-load-balancer`. You should see a clear spike at the time the script ran. The `HealthyHostCount` metric should remain flat at 2 throughout, confirming both instances stayed healthy under load.
