## user-data.sh — EC2 Bootstrap Script

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
