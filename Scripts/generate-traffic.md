## generate-traffic.sh — ALB Traffic Simulation

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
