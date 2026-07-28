# Architecture Specification & Network Topology

This document details the underlying network architecture, security boundaries, and routing infrastructure established for cross-VPC PostgreSQL migration.

---

## 1. Network Topology Overview

The infrastructure spans two independent Virtual Private Clouds (VPCs) in the same AWS Region (e.g., `us-east-1`).

```
+-------------------------------------------------------------------------+
|                              AWS REGION                                 |
|                                                                         |
|  +-------------------------------+   +-------------------------------+  |
|  |           VPC A               |   |           VPC B               |  |
|  |       (10.0.0.0/16)           |   |       (10.1.0.0/16)           |  |
|  |                               |   |                               |  |
|  |  +-------------------------+  |   |  +-------------------------+  |  |
|  |  |  Public Subnet A        |  |   |  |  Private Subnet B       |  |  |
|  |  |  (10.0.1.0/24)          |  |   |  |  (10.1.2.0/24)          |  |  |
|  |  |  - Internet Gateway     |  |   |  |  - Target RDS PG          |  |  |
|  |  |  - Bastion Host (EC2)   |  |   |  +-------------------------+  |  |
|  |  +-------------------------+  |   |                               |  |
|  |               |               |   |                               |  |
|  |  +------------v------------+  |   |                               |  |
|  |  |  Private Subnet A       |  |   |                               |  |
|  |  |  (10.0.2.0/24)          |  |   |                               |  |
|  |  |  - Source RDS PG        |  |   |                               |  |
|  |  +-------------------------+  |   +-------------------------------+  |
|  +---------------+---------------+                   ^                  |
|                  |                                   |                  |
|                  +======= VPC PEERING CONNECTION ====+                  |
|                           (pcx-peering-01)                              |
+-------------------------------------------------------------------------+
```

---

## 2. IP Address Management (IPAM) & Subnetting

| Network Component | IPv4 CIDR Block | Purpose | Associated Route Table |
| :--- | :--- | :--- | :--- |
| **VPC A** | `10.0.0.0/16` | Source Database Environment | `rtb-vpc-a-main` |
| **VPC A - Public Subnet** | `10.0.1.0/24` | Jumpbox / Bastion Host | `rtb-vpc-a-public` |
| **VPC A - Private Subnet 1** | `10.0.2.0/24` | Primary Source RDS PostgreSQL | `rtb-vpc-a-private` |
| **VPC A - Private Subnet 2** | `10.0.3.0/24` | Secondary AZ for DB Subnet Group | `rtb-vpc-a-private` |
| **VPC B** | `10.1.0.0/16` | Target Database Environment | `rtb-vpc-b-main` |
| **VPC B - Private Subnet 1** | `10.1.2.0/24` | Target RDS PostgreSQL | `rtb-vpc-b-private` |
| **VPC B - Private Subnet 2** | `10.1.3.0/24` | Secondary AZ for DB Subnet Group | `rtb-vpc-b-private` |

---

## 3. Route Tables Configuration

### Route Table: `rtb-vpc-a-public` (Public Subnet VPC-A)
- Destination: `0.0.0.0/0` -> Target: `igw-01a2b3c4` (Internet Gateway)
- Destination: `10.0.0.0/16` -> Target: `local`
- Destination: `10.1.0.0/16` -> Target: `pcx-peering-01` (VPC Peering Connection)

### Route Table: `rtb-vpc-a-private` (Private Subnet VPC-A)
- Destination: `10.0.0.0/16` -> Target: `local`
- Destination: `10.1.0.0/16` -> Target: `pcx-peering-01` (VPC Peering Connection)

### Route Table: `rtb-vpc-b-private` (Private Subnet VPC-B)
- Destination: `10.1.0.0/16` -> Target: `local`
- Destination: `10.0.0.0/16` -> Target: `pcx-peering-01` (VPC Peering Connection)

---

## 4. Security Group Configurations

### Bastion Security Group (`sg-bastion`)
- **Inbound Rules**:
  - Type: `SSH` | Protocol: `TCP` | Port Range: `22` | Source: `My-IP/32`
- **Outbound Rules**:
  - Type: `PostgreSQL` | Protocol: `TCP` | Port Range: `5432` | Destination: `10.0.0.0/16` & `10.1.0.0/16`

### Source RDS Security Group (`sg-source-rds`)
- **Inbound Rules**:
  - Type: `PostgreSQL` | Protocol: `TCP` | Port Range: `5432` | Source: `sg-bastion` (Bastion Host SG ID)
- **Outbound Rules**:
  - Standard AWS local outbound rules.

### Target RDS Security Group (`sg-target-rds`)
- **Inbound Rules**:
  - Type: `PostgreSQL` | Protocol: `TCP` | Port Range: `5432` | Source: `10.0.0.0/16` (VPC-A CIDR Block)
- **Outbound Rules**:
  - Standard AWS local outbound rules.
