# Security Architecture & Hardening Model

This document outlines the security controls and defense-in-depth measures implemented across network, compute, database, and IAM layers.

---

## 1. Network Boundary & Isolation

- **Non-Overlapping CIDRs**: VPC-A (`10.0.0.0/16`) and VPC-B (`10.1.0.0/16`) do not overlap, enabling strict subnetwork segmentation.
- **Private Subnets Only for Storage**: Databases are provisioned in private subnets with `Publicly Accessible` flag set to `False`. Neither RDS instance possesses a public IP or direct path to the Internet.
- **Controlled Ingress**: Traffic enters VPC-A exclusively via SSH on port 22 through a designated Bastion Host protected by Security Groups restricted to a specific administrator IP `/32`.

---

## 2. Security Group Matrix (Least Privilege Access)

```
[ Workstation IP ] ──(SSH:22)──> [ Bastion Host SG ] ──(PG:5432)──> [ Source RDS SG ]
                                          │
                                   (VPC Peering:5432)
                                          │
                                          ▼
                                 [ Target RDS SG ]
```

### Ingress & Egress Rules Breakdown

1. **Bastion Host Security Group (`sg-bastion`)**:
   - Inbound: `TCP/22` restricted to admin IP address.
   - Outbound: `TCP/5432` to `10.0.0.0/16` and `10.1.0.0/16`.

2. **Source RDS Security Group (`sg-source-rds`)**:
   - Inbound: `TCP/5432` allowed **only** from Security Group `sg-bastion`.
   - Egress: Disabled except default VPC routing.

3. **Target RDS Security Group (`sg-target-rds`)**:
   - Inbound: `TCP/5432` allowed **only** from VPC-A CIDR range (`10.0.0.0/16`) or `sg-bastion`.
   - Egress: Disabled except default VPC routing.

---

## 3. Data Protection & Encryption

- **Encryption at Rest**: Storage volumes encrypted using standard AWS KMS Service Managed Keys (`aws/rds`).
- **Encryption in Transit**: Connections between PostgreSQL clients and Amazon RDS instances require TLS/SSL (`sslmode=require`).
- **Credential Protection**: Database passwords passed securely via environment variables or interactive CLI prompts rather than plaintext scripts.

---

## 4. Key Management & IAM Policy

- **Service-Linked Roles**: AWS RDS Service-Linked Role utilized for CloudWatch publishing.
- **No Hardcoded Credentials**: Secrets, private SSH keys (`.pem`), and AWS tokens are explicitly excluded from version control using `.gitignore`.
