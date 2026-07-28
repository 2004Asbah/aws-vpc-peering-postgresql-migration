# Implementation Blueprint & Execution Guide

This document contains step-by-step instructions to set up the VPC Peering, Bastion host, RDS databases, and run the database migration.

---

## Prerequisites

- Active AWS Account with Free Tier eligibility.
- AWS CLI configured or Access to AWS Management Console.
- SSH client (Git Bash or OpenSSH).
- Basic familiarity with Linux CLI and PostgreSQL administration commands.

---

## Phase 1: Network Infrastructure & VPC Peering Setup

1. **Create VPC A (Source Network)**:
   - Name tag: `VPC-A-Source`
   - IPv4 CIDR block: `10.0.0.0/16`
   - Subnets:
     - Public Subnet: `10.0.1.0/24` (Availability Zone A)
     - Private Subnet 1: `10.0.2.0/24` (Availability Zone A)
     - Private Subnet 2: `10.0.3.0/24` (Availability Zone B)
   - Create and attach Internet Gateway (`IGW-VPC-A`) to Public Subnet.

2. **Create VPC B (Target Network)**:
   - Name tag: `VPC-B-Target`
   - IPv4 CIDR block: `10.1.0.0/16`
   - Subnets:
     - Private Subnet 1: `10.1.2.0/24` (Availability Zone A)
     - Private Subnet 2: `10.1.3.0/24` (Availability Zone B)

3. **Establish VPC Peering Connection**:
   - Go to **VPC Console -> Peering Connections**.
   - Create Peering Connection:
     - Requester: `VPC-A-Source` (`10.0.0.0/16`)
     - Accepter: `VPC-B-Target` (`10.1.0.0/16`)
   - Accept Peering Request under **Actions -> Accept Request**.

4. **Update Route Tables**:
   - Add route in VPC-A Route Tables:
     `Destination: 10.1.0.0/16` -> `Target: pcx-xxxxxx`
   - Add route in VPC-B Route Tables:
     `Destination: 10.0.0.0/16` -> `Target: pcx-xxxxxx`

---

## Phase 2: Bastion Host Provisioning

1. Launch an Amazon Linux 2 or Ubuntu 22.04 LTS EC2 Instance in `VPC-A Public Subnet`.
2. Instance Type: `t2.micro` or `t3.micro` (Free Tier).
3. Security Group:
   - Inbound: SSH (Port 22) from your local workstation IP (`x.x.x.x/32`).
4. SSH into the Bastion Instance and install PostgreSQL client utilities:
   ```bash
   sudo dnf update -y
   sudo dnf install -y postgresql15 postgresql16 nc
   ```

---

## Phase 3: Amazon RDS Database Deployment

1. **RDS Subnet Groups**:
   - Create Subnet Group `rds-subnets-a` selecting Private Subnets in VPC-A.
   - Create Subnet Group `rds-subnets-b` selecting Private Subnets in VPC-B.

2. **Deploy Source RDS (`postgres17-source`)**:
   - Engine: PostgreSQL 17 or 18 (Free Tier).
   - DB Instance Class: `db.t3.micro` / `db.t4g.micro`.
   - VPC: `VPC-A-Source`.
   - DB Subnet Group: `rds-subnets-a`.
   - Publicly Accessible: **No**.
   - Monitoring: **Database Insights - Standard (7 days retention)**. Disable Enhanced Monitoring.

3. **Deploy Target RDS (`postgres18-target`)**:
   - Engine: PostgreSQL 18.
   - DB Instance Class: `db.t3.micro` / `db.t4g.micro`.
   - VPC: `VPC-B-Target`.
   - DB Subnet Group: `rds-subnets-b`.
   - Publicly Accessible: **No**.
   - Monitoring: **Database Insights - Standard (7 days retention)**.

---

## Phase 4: Database Migration Execution

1. **Connect to Bastion Host**:
   ```bash
   ssh -i ~/.ssh/your-key.pem ec2-user@<bastion-public-ip>
   ```

2. **Verify Connectivity to Source and Target Database Instances**:
   ```bash
   nc -zv <source-rds-endpoint> 5432
   nc -zv <target-rds-endpoint> 5432
   ```

3. **Export Source Database**:
   ```bash
   pg_dump -h <source-rds-endpoint> -U postgres -d company_db -F c -b -v -f company_db.dump
   ```

4. **Import into Target Database**:
   ```bash
   pg_restore -h <target-rds-endpoint> -U postgres -d company_db -v company_db.dump
   ```

5. **Verify Row Counts & Table Integrity**:
   Run schema comparison queries (detailed in `commands/sql-commands.md`).

6. **Trigger Slack Notification Payload**:
   ```bash
   curl -X POST -H 'Content-type: application/json' \
     --data '{"text":"🎉 AWS VPC Peering PostgreSQL Migration completed successfully!"}' \
     https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
   ```
