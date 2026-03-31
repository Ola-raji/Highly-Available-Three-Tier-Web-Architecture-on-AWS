# verify-nginx.sh — Post-Launch Nginx Verification

Runs a set of checks on a freshly launched EC2 instance to confirm that Nginx installed correctly, the custom AZ page is being served, outbound internet works through the NAT Gateway, and instance metadata is being fetched correctly via IMDSv2. Used after launch to validate the User Data script ran successfully before registering the instance with the ALB.

**Where to run:** On each EC2 instance via SSM Session Manager, as root.  
**When to run:** After the instance reaches Running state and before confirming it healthy in the target group.

---

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

---

## Notes

**What each check confirms:**

| Check | What it proves |
|---|---|
| Nginx service status | User Data ran successfully and the service started |
| Local HTTP response | Custom HTML page was written correctly with AZ and instance ID populated |
| Outbound internet | NAT Gateway is reachable and the private route table is configured correctly |
| Instance metadata | IMDSv2 token fetch works and metadata is accessible — confirms the Launch Template IMDSv2 setting is not blocking access |

**The external IP is the NAT Gateway's Elastic IP — not the instance's IP.**  
This is NAT in action. The instance has no public IP of its own. All outbound traffic exits through the NAT Gateway, which substitutes its own public Elastic IP as the source address. This is what allows private instances to reach the internet for package installs and API calls without being directly reachable inbound.
