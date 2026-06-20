# Day 01 — Single VM Web Application
**Level:** 1 of 10 | **Company:** Contoso Boutique

## Business Problem
A small furniture retailer needed a secure e-commerce site live
in 6 weeks with no in-house infrastructure team and a sub-$300/month
budget.

## Architecture
![Architecture Diagram](architecture-diagram/day01-architecture.png)

## Azure Services Used
| Service | Purpose |
|---|---|
| Azure VM (Standard_B2s) | Web application host |
| Azure SQL Database (S1) | Managed relational database |
| Azure Bastion | Secure management access |
| Key Vault | Secret and certificate storage |
| Storage Account | Backup and diagnostic logs |
| Log Analytics | Centralized logging |
| Azure Monitor | Alerting and metrics |

## Key Design Decisions
- Managed Identity used for zero-credential Key Vault access
- Database restricted to VNet only — no public endpoint
- No public SSH/RDP ports — Bastion only
- RBAC scoped to Resource Group, not Subscription

## Network Design
- VNet: 10.0.0.0/16
- web-subnet: 10.0.1.0/24
- AzureBastionSubnet: 10.0.2.0/27
- NSG: allow 443 inbound, allow 22 from Bastion only, deny all else

## Security Highlights
- System-Assigned Managed Identity on VM
- Key Vault RBAC mode (not legacy access policies)
- SQL Database: public network access disabled
- DDoS Infrastructure protection (free tier)

## High Availability
- Single VM — no redundancy (deliberate at this scale)
- RTO: 8 hours | RPO: 24 hours
- Azure SQL automated backups (7-day PITR)

## Cost Estimate
| Resource | Monthly Cost |
|---|---|
| VM (Standard_B2s) | ~$60 |
| Managed Disk (64GB) | ~$10 |
| Azure SQL (S1) | ~$30 |
| Azure Bastion | ~$140 |
| Storage + KV + LA | ~$9 |
| **Total** | **~$249/mo** |

## Lab Validation
- [ ] Direct SSH to VM public IP fails (times out)
- [ ] Bastion connection succeeds
- [ ] Key Vault secret retrieved via Managed Identity
- [ ] SQL unreachable from outside VNet
- [ ] CPU alert fires under load test

## Challenges Faced
<!-- Fill in after completing the lab -->

## Lessons Learned
<!-- Fill in after completing the lab -->

## Interview Story
See [lesson-notes.md](lesson-notes.md) — Section 14D

## Resume Bullets
- Deployed secure single-tier Azure architecture using VM, SQL,
  Key Vault, and Bastion with zero public management port exposure
- Implemented Managed Identity for credential-less Key Vault access
- Isolated Azure SQL Database to VNet-only connectivity
- Configured Azure Monitor alerting for proactive CPU monitoring

## What I Would Add in Production
- CI/CD pipeline replacing manual SSH deployment
- Application Gateway + WAF
- Azure Backup for the VM
- PIM for just-in-time admin access

## Next
[Day 02 — Load Balanced Architecture →](../day02-load-balanced/README.md)
