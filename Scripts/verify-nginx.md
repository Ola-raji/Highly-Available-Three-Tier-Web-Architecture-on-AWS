## verify-nginx.sh — Post-Launch Nginx Verification

```bash
#!/bin/bash
# Nginx Verification Script
# Run on EC2 instance via SSM Session Manager after launch
# Confirms Nginx is running and serving the custom AZ page

echo "=== Nginx Service Status ==="
systemctl status nginx --no-pager

echo ""
echo "=== Local HTTP Response ==="
curl -s http://localhost | grep -E "(Zone|Instance|healthy)"

echo ""
echo "=== Outbound Internet (NAT Gateway test) ==="
EXTERNAL_IP=$(curl -s https://checkip.amazonaws.com)
echo "External IP (NAT Gateway): $EXTERNAL_IP"

echo ""
echo "=== Instance Metadata ==="
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
echo "AZ: $AZ"
echo "Instance ID: $INSTANCE_ID"
```

---

## Expected Output

```
=== Nginx Service Status ===
Active: active (running)

=== Local HTTP Response ===
Availability Zone: <strong>us-east-1a</strong>
Instance ID: i-0xxxxxxxxxxxxxxxxx
This server is healthy and running Nginx

=== Outbound Internet (NAT Gateway test) ===
External IP (NAT Gateway): 54.x.x.x

=== Instance Metadata ===
AZ: us-east-1a
Instance ID: i-0xxxxxxxxxxxxxxxxx
```
