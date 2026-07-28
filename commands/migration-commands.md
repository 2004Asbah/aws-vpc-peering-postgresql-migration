# PostgreSQL Migration & CLI Commands Reference

This reference cheat sheet contains all command-line operations executed during the database migration.

---

## 1. Network & Port Verification Commands

Run on the Bastion Host to confirm connectivity before executing dump/restore:

```bash
# Test SSH connectivity to Bastion Host
ssh -i ~/.ssh/bastion-key.pem ec2-user@<BASTION_PUBLIC_IP>

# Test network path to Source RDS PostgreSQL port 5432
nc -zv -w 5 <SOURCE_RDS_ENDPOINT> 5432

# Test network path across VPC Peering to Target RDS PostgreSQL port 5432
nc -zv -w 5 <TARGET_RDS_ENDPOINT> 5432
```

---

## 2. PostgreSQL Dump Commands

Execute on the Bastion Host to extract schema and dataset:

```bash
# Set password variable to prevent interactive prompt in automated scripts
export PGPASSWORD="YourSecurePasswordHere"

# Perform compressed custom format dump of source database
pg_dump \
  -h postgres17-source.c123456789.us-east-1.rds.amazonaws.com \
  -U postgres \
  -d company_db \
  -F c \
  -b \
  -v \
  -f company_db_backup.dump

# Verify dump file size and format
ls -lh company_db_backup.dump
file company_db_backup.dump
```

---

## 3. PostgreSQL Target Database Creation & Restore

```bash
# Connect to Target RDS default postgres database and create destination database
psql \
  -h postgres18-target.c123456789.us-east-1.rds.amazonaws.com \
  -U postgres \
  -d postgres \
  -c "CREATE DATABASE company_db;"

# Perform parallelized restore using pg_restore
pg_restore \
  -h postgres18-target.c123456789.us-east-1.rds.amazonaws.com \
  -U postgres \
  -d company_db \
  -v \
  --no-owner \
  --no-acl \
  company_db_backup.dump
```

---

## 4. Slack Notification Automation Script

```bash
#!/bin/bash
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/YOUR_WORKSPACE_ID/YOUR_CHANNEL_ID/YOUR_WEBHOOK_TOKEN"

PAYLOAD=$(cat <<EOF
{
  "attachments": [
    {
      "color": "#36a64f",
      "title": "✅ AWS PostgreSQL Database Migration Succeeded",
      "fields": [
        { "title": "Source VPC", "value": "VPC-A (10.0.0.0/16)", "short": true },
        { "title": "Target VPC", "value": "VPC-B (10.1.0.0/16)", "short": true },
        { "title": "Peering Connection", "value": "pcx-peering-01 (Active)", "short": true },
        { "title": "Database", "value": "company_db", "short": true }
      ],
      "footer": "AWS Migration Automation",
      "ts": $(date +%s)
    }
  ]
}
EOF
)

curl -X POST -H 'Content-type: application/json' --data "$PAYLOAD" "$SLACK_WEBHOOK_URL"
```
