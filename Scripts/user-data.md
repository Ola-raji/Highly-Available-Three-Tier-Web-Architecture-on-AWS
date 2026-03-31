# user-data.sh — EC2 Bootstrap Script

Runs automatically as root on first boot via EC2 User Data. Installs Nginx, fetches instance metadata using IMDSv2, and writes a custom HTML page showing the instance's Availability Zone and ID. This makes ALB traffic distribution visually observable in the browser.

**When it runs:** At EC2 launch — automatically, once, as root.  
**Where it's used:** Pasted into the User Data field under Advanced Details when launching EC2 instances, and stored in the Launch Template for ASG-managed instances.

---

```bash
#!/bin/bash
# EC2 User Data Bootstrap Script
# Amazon Linux 2023 — IMDSv2 compatible
# Installs Nginx and creates a custom index page showing AZ and instance ID

yum update -y
yum install -y nginx
systemctl start nginx
systemctl enable nginx

# IMDSv2 token request
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Fetch instance metadata using token
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)

# Write custom index page
cat > /usr/share/nginx/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head><title>HA Project</title></head>
<body style="font-family:Arial;text-align:center;padding:60px;background:#f0f4f8">
  <h1 style="color:#1a56db">High Availability Demo</h1>
  <p style="font-size:20px">Availability Zone: <strong>$AZ</strong></p>
  <p style="font-size:16px;color:#555">Instance ID: $INSTANCE_ID</p>
  <p style="color:#28a745">This server is healthy and running Nginx</p>
</body>
</html>
EOF
```

---

## Notes

**Why IMDSv2?**  
Amazon Linux 2023 enforces IMDSv2 by default. The original IMDSv1 curl calls (`curl http://169.254.169.254/...`) fail silently — returning an empty string with no error. The fix is to first request a session token, then pass that token as a header on every subsequent metadata call. IMDSv2 is a security improvement that prevents SSRF attacks from leaking instance credentials via the metadata endpoint.

**Why no `sudo`?**  
User Data scripts run as root at launch — `sudo` is not needed. The same commands run manually in an SSM Session Manager terminal require `sudo su -` first, because SSM drops you in as the unprivileged `ssm-user`.

**Why `systemctl enable nginx`?**  
`start` launches Nginx now. `enable` ensures it starts automatically if the instance reboots. Both are needed for a production-ready setup.
