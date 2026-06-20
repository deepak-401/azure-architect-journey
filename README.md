# Azure Architect Journey

A progressive, scenario-driven Azure architecture portfolio built across
50+ real-world lessons spanning 10 difficulty levels — from single-VM
deployments to Fortune 500-scale multi-region enterprise architectures.

## What This Is

Each lesson follows a consistent structure:
- Real business scenario with a named company (Contoso Boutique and others)
- Requirements gathering (functional, security, compliance, cost)
- Architecture diagrams
- Deep dives: networking, security, high availability, monitoring, cost
- Terraform resource hierarchy and IaC
- Interview preparation (AZ-104, AZ-500, AZ-700, AZ-305)
- Hands-on lab with validation steps
- Portfolio documentation with ADRs, runbooks, and STAR stories

## Certification Targets
- AZ-104 Azure Administrator
- AZ-500 Azure Security Engineer
- AZ-700 Azure Network Engineer
- AZ-305 Azure Solutions Architect Expert

## Architecture Levels

| Level | Focus | Status |
|---|---|---|
| 1 | Beginner | ✅ Days 1–2 complete |
| 2 | Small Business | 🔄 Days 3–4 complete, continuing |
| 3 | Enterprise | 🔜 Upcoming |
| 4 | Multi-Region | 🔜 Upcoming |
| 5 | Highly Secure | 🔜 Upcoming |
| 6 | Hybrid | 🔜 Upcoming |
| 7 | FinOps | 🔜 Upcoming |
| 8 | AKS | 🔜 Upcoming |
| 9 | AI & Data | 🔜 Upcoming |
| 10 | Fortune 500 | 🔜 Upcoming |

## Company Narrative: Contoso Boutique

A furniture e-commerce retailer whose architecture evolves lesson by
lesson in direct response to real business problems — not arbitrary
complexity additions.

| Day | Problem Solved | Key Services |
|---|---|---|
| 1 | Launch a secure web app with zero on-prem infra | VM, SQL, Bastion, Key Vault |
| 2 | Eliminate downtime during patching | Standard LB, Availability Zones |
| 3 | Fix configuration drift, add CI/CD | App Service, Deployment Slots, App Gateway, WAF |
| 4 | Database resilience against regional failure | SQL Failover Group, Geo-Replication |

## Skills Demonstrated
- Network segmentation (VNets, subnets, NSGs, UDRs)
- Zero-credential architecture (Managed Identity, Key Vault)
- High availability (Availability Zones, Failover Groups, Slot Swaps)
- Security layering (WAF, Bastion, RBAC, PIM)
- Infrastructure as Code (Terraform)
- CI/CD (GitHub Actions, Deployment Slots)
- Disaster recovery (RTO/RPO engineering, runbooks)
- Cost optimization (Reserved Instances, right-sizing, FinOps)

## How to Navigate
Each lesson folder contains its own README, architecture diagram,
Terraform code, and interview preparation material.
Start at contoso-boutique/day01-single-vm/README.md.
