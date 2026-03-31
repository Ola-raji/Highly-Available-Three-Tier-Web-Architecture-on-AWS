# db-connection-test.sh — RDS Connectivity Verification

Verifies that an EC2 instance can reach the RDS database through the security group chain. Connects to the RDS endpoint using the MySQL client and runs a set of basic SQL commands to confirm the database is reachable, the schema exists, and read/write operations work correctly.

**Where to run:** On any ASG-managed EC2 instance via SSM Session Manager, as root.  
**When to run:** After the RDS instance status shows Available and the `ha-rds-sg` security group is confirmed to allow port 3306 from `ha-ec2-sg`.

---

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

---

## Notes

**Why `mariadb105` instead of `mysql`?**  
Amazon Linux 2023 removed the `mysql` package from its default repositories. `mariadb105` is a fully compatible drop-in replacement for the MySQL client — the connection syntax, commands, and SQL dialect are identical. The server running on RDS is still MySQL 8.0; only the client tool changes.

**Why is the password not in the script?**  
Hardcoding credentials in a script is a security risk, particularly in a version-controlled repository. The `-p` flag without a value causes MySQL to prompt for the password interactively — nothing is stored in shell history or visible on screen while typing.

**What a successful connection confirms:**  
The security group chain is working correctly — `ha-rds-sg` is accepting port 3306 connections from `ha-ec2-sg`. The RDS instance is reachable from within the private subnet. The database schema `haprojectdb` was created at launch. The full three-tier architecture is end-to-end functional.

**If the connection fails:**  
The most common cause is an incorrect security group source. Verify that `ha-rds-sg` inbound rule references `ha-ec2-sg` as the source (not a CIDR range), and that the RDS instance is in Available state rather than still Creating or Modifying.
