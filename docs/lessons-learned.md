# Key Takeaways & Technical Lessons Learned

This document records architectural insights and operational takeaways gained during this AWS VPC Peering & Database Migration project.

---

## 1. AWS RDS Engine Versioning & Extended Support Costs

* **Insight**: AWS charges substantial hourly Extended Support fees for older PostgreSQL versions (such as PG 10, 11, and 12).
* **Takeaway**: Always use modern engine versions (PostgreSQL 17 or PostgreSQL 18) for Free Tier lab environments to ensure zero billing charges.

---

## 2. Cross-VPC Peering DNS Resolution

* **Insight**: By default, resolving RDS endpoints across VPC Peering connections requires enabling DNS Hostnames and DNS Resolution flags on both VPCs.
* **Takeaway**: Enabling `enableDnsHostnames` and `enableDnsSupport` on both VPCs ensures native AWS private DNS names resolve correctly from the Bastion host to both source and target RDS endpoints.

---

## 3. Storage & Network Performance Optimizations

* **Insight**: `pg_dump` with custom directory archive format (`-F c`) compresses output files significantly and speeds up subsequent multi-threaded `pg_restore` operations.
* **Takeaway**: Using directory format (`-F c` or `-F d`) and parallel restore jobs (`pg_restore -j 4`) decreases downtime during database migration windows.

---

## 4. Operational Hygiene & Cost Discipline

* **Insight**: Forgetting to turn off CloudWatch Enhanced Monitoring or selecting Database Insights - Advanced can inadvertently accrue CloudWatch charges even on Free Tier RDS instances.
* **Takeaway**: Rigorous auditing of default dropdowns during resource creation is essential to maintaining absolute zero-cost cloud architecture lab setups.
