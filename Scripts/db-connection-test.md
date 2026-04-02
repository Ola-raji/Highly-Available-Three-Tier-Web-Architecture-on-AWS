## db-connection-test.sh — RDS Connectivity Verification

```bash
#!/bin/bash
# RDS Connection Test Script
# Run from EC2 instance via SSM Session Manager (as root)
# Verifies EC2 → RDS connectivity through security group chain
#
# Prerequisites:
#   - MySQL client installed (yum install -y mariadb105)
#   - RDS endpoint available
#   - ha-rds-sg allows port 3306 from ha-ec2-sg
#
# Usage: Replace RDS_ENDPOINT with your actual endpoint before running

RDS_ENDPOINT="YOUR-RDS-ENDPOINT.rds.amazonaws.com"

echo "Installing MySQL client..."
yum install -y mariadb105 2>/dev/null

echo ""
echo "Connecting to: $RDS_ENDPOINT"
echo "Enter RDS master password when prompted..."

mysql -h "$RDS_ENDPOINT" -u admin -p haprojectdb
```

Once connected at the `mysql>` prompt, run these commands one at a time:

```sql
-- Confirm the database exists
SHOW DATABASES;

-- Switch into the project database
USE haprojectdb;

-- Create a test table
CREATE TABLE servers (
  id INT AUTO_INCREMENT PRIMARY KEY,
  hostname VARCHAR(100),
  az VARCHAR(20),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert test records
INSERT INTO servers (hostname, az) VALUES ('ha-web-server-1a', 'us-east-1a');
INSERT INTO servers (hostname, az) VALUES ('ha-web-server-1b', 'us-east-1b');

-- Read back to confirm persistence
SELECT * FROM servers;

-- Exit
EXIT;
```
