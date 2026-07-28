# Troubleshooting Guide & Common Pitfalls

This document details issues encountered during cross-VPC database migrations and their solutions, alongside cost protection rules.

---

## 1. Network & Routing Bottlenecks

### Issue A: `nc -zv <target-rds-endpoint> 5432` times out
* **Root Cause**: Missing route entry in VPC-B route table or restrictive inbound rules in Target RDS Security Group.
* **Resolution**:
  1. Verify VPC-B's private route table contains `10.0.0.0/16 -> pcx-xxxxxx`.
  2. Check VPC Peering Status: Ensure Peering Connection state is **Active** (Accept request must be confirmed).
  3. Verify Target RDS Security Group allows TCP port 5432 inbound from VPC-A CIDR (`10.0.0.0/16`) or Bastion Security Group.

### Issue B: Bastion Host cannot resolve RDS Endpoint hostname
* **Root Cause**: `enableDnsHostnames` or `enableDnsSupport` is disabled on the VPC.
* **Resolution**:
  1. Open VPC Console -> Select VPC-A -> Edit VPC Settings.
  2. Enable **DNS Hostnames** and **DNS Support**.
  3. Perform the same check on VPC-B.

---

## 2. PostgreSQL Authentication & Version Issues

### Issue C: `pg_dump: error: connection to server at ... failed: FATAL: password authentication failed`
* **Root Cause**: Incorrect master username or password supplied during command execution.
* **Resolution**:
  - Test login with explicit password prompt:
    ```bash
    psql -h <source-rds-endpoint> -U postgres -d postgres -W
    ```
  - If password is forgotten, modify DB master password via AWS RDS Console -> Modify instance (applies immediately).

### Issue D: `pg_restore: error: server version mismatch`
* **Root Cause**: Restoring from a newer PostgreSQL version to an older PostgreSQL version.
* **Resolution**: Always ensure Target PostgreSQL engine version is equal to or greater than Source PostgreSQL version (e.g. Source PG 17 -> Target PG 18).

---

## 3. AWS Free Tier Billing Avoidance & Resource Cleanup

To guarantee **$0.00** charges, verify that no paid features were enabled during setup:

| RDS Feature | Safe Free Tier Setting | Risky / Paid Setting (DO NOT USE) |
| :--- | :--- | :--- |
| **Database Insights** | `Database Insights - Standard` (7 days) | `Database Insights - Advanced` (15 months retention) |
| **Performance Insights** | `Standard retention (7 days)` | `24 months retention` |
| **Enhanced Monitoring** | `Disabled` | `Enabled (60s / 1s granularity)` |
| **DevOps Guru** | `Disabled` | `Enabled` |
| **Multi-AZ Deployment** | `Single-AZ` | `Multi-AZ Deployment` |
| **PG Engine Version** | `PG 17 / PG 18` | `PG 10 / PG 11 / PG 12 (Extended Support Fee)` |

---

## 4. Complete Resource Deletion Checklist (Clean-Up Order)

To prevent ongoing charges, delete all resources in this precise order:

1. **EC2 Instances**: Terminate Bastion Host instance.
2. **Elastic IPs**: Disassociate and release any unattached Elastic IPs.
3. **RDS Databases**:
   - Delete `postgres18-target` (Uncheck *Create final snapshot*, Uncheck *Retain automated backups*).
   - Delete `postgres17-source`.
4. **RDS DB Subnet Groups**: Delete custom DB subnet groups once instances are terminated.
5. **VPC Peering Connection**: Delete `pcx-peering-01`.
6. **Route Tables**: Remove peering routes from VPC route tables.
7. **Internet Gateways**: Detach and delete Internet Gateway attached to VPC-A.
8. **Subnets & VPCs**: Delete subnets, then delete VPC-A and VPC-B.
9. **Security Groups**: Delete custom security groups.
