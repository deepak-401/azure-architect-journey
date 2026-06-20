# Day 1 — Level 1: Beginner Architecture
## Lesson: Contoso Boutique — Single-VM Web App with Azure SQL Database

> Training Path Position: Level 1 of 10 | Day 1
> Certification focus today: AZ-104 (60%), AZ-500 (15%), AZ-700 (15%), AZ-305 (10%)

---

## 1. Business Scenario

**Company Background**
Contoso Boutique is a 40-person specialty furniture retailer based in Austin, Texas. They run three physical stores and have historically taken orders by phone and a basic on-premises file-server-driven spreadsheet system. They have no in-house infrastructure team — just one IT generalist.

**Business Problem**
The owner wants to launch a customer-facing e-commerce site (product catalog + order placement) within 6 weeks for the holiday season. Their on-prem server is a 2017 Dell tower in a back office with no redundancy, no backup testing, and a residential internet connection. A single hardware failure would take the business offline for days.

**Why Azure**
- No capital budget for a new on-prem server or data center colocation.
- Need to go live in weeks, not months — no time for procurement cycles.
- Want pay-as-you-go economics since order volume is unproven (could be 50 orders/day or 5,000).
- Owner wants "someone else" responsible for physical hardware, power, and disk failures.
- Azure free trial + small reserved spend fits a sub-$300/month IT budget for Year 1.

This is intentionally the **simplest possible production-realistic architecture** — a single VM and a managed database. Every later lesson adds the pieces a real architect would add as the business grows (load balancing, WAF, multi-region, AKS, etc.). Don't skip this level even if it feels "too easy" — production outages most often happen because someone skipped fundamentals like NSG rules, backup retention, or disk sizing.

---

## 2. Requirements Gathering

**Functional Requirements**
- Public website serving a product catalog (≈200 SKUs) and shopping cart.
- Order data persisted in a relational database.
- Admin can SSH/RDP in to deploy code updates manually (no CI/CD yet — that's Level 2).
- Daily automated backup of the database.

**Non-Functional Requirements**
- Support ~200 concurrent users at peak (holiday traffic), ~20 baseline.
- Page load under 2 seconds for catalog browsing.
- 99.9% uptime target (not 99.99% — owner accepts brief downtime for patching at this stage; this is a deliberate, documented trade-off, not an oversight).

**Security Requirements**
- No direct internet exposure of management ports (RDP/SSH).
- Database not publicly reachable from the internet.
- Encryption at rest for the database and disks (Azure default, but must be explicitly verified, never assumed).
- Principle of least privilege for the one admin account.

**Compliance Requirements**
- Standard PCI-DSS awareness: **no card numbers are stored or processed by Contoso's own systems** — payment will be handled by a third-party processor (Stripe/Square) in a later lesson. This is a deliberate scope boundary: building your own card storage is a compliance trap to avoid, not a feature to build.
- Basic data residency: customer data stays in a US Azure region.

**Cost Requirements**
- Target: under $150/month for compute + database during steady state, acceptable to spike to ~$250 during the holiday peak month.
- No reserved instance commitment yet (workload is unproven — committing to a 1-year RI before you know your usage pattern is a common beginner mistake).

---

## 3. Architecture Diagram

```
                              Internet
                                 |
                                 |  HTTPS (443) only
                                 v
                      +-----------------------+
                      |   Public IP (Static)  |
                      +-----------------------+
                                 |
                      +-----------------------+
                      |   Network Security    |
                      |   Group (NSG)          |
                      |  Allow: 443 (Internet) |
                      |  Allow: 22 (Bastion)   |
                      |  Deny:  All else       |
                      +-----------------------+
                                 |
        VNet: 10.0.0.0/16  ------+------------------------------
        |                                                       |
        |  Subnet: web-subnet (10.0.1.0/24)                     |
        |   +------------------------------+                    |
        |   |   Azure VM (Ubuntu 22.04)    |                    |
        |   |   Standard_B2s                |                    |
        |   |   - Nginx + Node.js app       |                    |
        |   |   - Managed Identity enabled  |                    |
        |   +------------------------------+                    |
        |              |   Private link / VNet rule              |
        |              v                                         |
        |   +------------------------------+                    |
        |   |   Azure SQL Database          |                    |
        |   |   (PaaS, Standard S1 tier)    |                    |
        |   |   Firewall: VNet rule only    |                    |
        |   +------------------------------+                    |
        |                                                       |
        |  Subnet: bastion-subnet (10.0.2.0/27)                  |
        |   +------------------------------+                    |
        |   |  Azure Bastion                |                    |
        |   +------------------------------+                    |
        -------------------------------------------------------

                  +-----------------------------+
                  |  Storage Account (LRS)       |
                  |  - SQL automated backups     |
                  |  - VM diagnostic logs         |
                  +-----------------------------+

                  +-----------------------------+
                  |  Key Vault                    |
                  |  - DB connection string       |
                  |  - TLS cert (App)             |
                  +-----------------------------+

                  +-----------------------------+
                  |  Log Analytics Workspace      |
                  |  + Azure Monitor              |
                  +-----------------------------+
```

Note what's deliberately **absent** at Level 1: no Load Balancer (single VM, nothing to balance yet), no Application Gateway/WAF (added Level 2 once there's real internet exposure risk), no multi-region (added Level 4). A senior architect doesn't add complexity the business hasn't earned yet — every component should map to a stated requirement above.

---

## 4. Service Explanation

### Azure Virtual Machine (Standard_B2s)
- **What it does:** IaaS compute — a full Linux OS instance running your web app (Nginx reverse-proxying a Node.js process).
- **Why used:** Fastest path to "lift and run" custom code without refactoring into a PaaS app model; team has zero App Service experience but knows Linux administration.
- **Why alternatives weren't chosen:** Azure App Service would be the "correct" production answer for a stateless web app (less patching burden, built-in scaling) — but the requirement explicitly states the owner wants hands-on SSH access for now during the learning phase. We document this as **technical debt to revisit in Level 2**, not as an ideal choice.
- **Pricing considerations:** B-series is "burstable" — cheap baseline CPU credit model, ideal for spiky small workloads instead of paying for constant D-series performance you don't need at 20 concurrent users.
- **Common interview questions:**
  - "What's the difference between B-series and D-series VMs, and when would B-series cause a production problem?" (Answer: CPU credit exhaustion under sustained load → throttling.)
  - "How do you resize a VM with minimal downtime?"
  - "What's the difference between Azure VM availability via Availability Set vs Availability Zone?"

### Azure SQL Database (PaaS, Standard S1)
- **What it does:** Fully managed relational database — Microsoft handles patching, backups, HA infrastructure.
- **Why used:** No DBA on staff; PaaS removes OS/engine patching entirely. Automated backups satisfy the backup requirement out of the box.
- **Why alternatives weren't chosen:** SQL Server on a VM (IaaS) would require the team to manage patching/backups themselves — directly against the "no in-house infra team" constraint. Cosmos DB was considered and rejected: the data is relational (orders, line items, customers) with no need for global distribution or flexible schema at this stage.
- **Pricing considerations:** DTU-based Standard tier is simpler to budget for low-traffic workloads than vCore; can switch to vCore + Reserved Capacity later once usage patterns are known (covered in Level 7 FinOps).
- **Common interview questions:**
  - "DTU vs vCore purchasing model — when do you pick each?"
  - "How does Azure SQL Database achieve high availability under the hood?" (Answer: it's built on Always On Availability Groups internally, abstracted away.)
  - "How do you restrict Azure SQL Database to only be reachable from a specific VNet?"

### Storage Account (LRS)
- **What it does:** Object storage backing automated DB backup retention and VM boot diagnostics/logs.
- **Why used:** Cheapest redundancy tier (LRS = 3 copies in one datacenter) is sufficient — this is non-customer-facing operational data, not the primary system of record.
- **Why alternatives weren't chosen:** GRS (geo-redundant) was considered and explicitly deferred — it roughly doubles storage cost and the business has no DR requirement yet at Level 1. This will be revisited in Level 4.
- **Common interview questions:**
  - "LRS vs ZRS vs GRS vs RA-GRS — explain the differences and a scenario for each."

### Key Vault
- **What it does:** Centralized secret/cert storage with access-policy or RBAC-based control.
- **Why used:** The app must never have the DB connection string or TLS cert hardcoded in source code or a config file on disk.
- **Common interview questions:**
  - "How does a VM authenticate to Key Vault without storing a credential anywhere?" (Answer: Managed Identity.)

### Azure Bastion
- **What it does:** Provides browser-based SSH/RDP to VMs without exposing port 22/3389 to the public internet.
- **Why used:** Directly satisfies the security requirement "no direct internet exposure of management ports."
- **Why alternatives weren't chosen:** A traditional jump-box VM would itself need patching and its own public IP exposure risk; Bastion is a managed PaaS service with no attack surface to maintain.
- **Pricing considerations:** Bastion has an hourly cost even when idle — at Level 1 scale this is a real, debatable cost trade-off (~$140/month for Basic SKU) vs. just using a tightly-scoped NSG rule + VPN. We chose Bastion here for the **security-first learning habit**; in Section 13 we'll discuss when an architect would instead use a temporary just-in-time (JIT) NSG rule via Defender for Cloud to cut this cost.

---

## 5. Networking Deep Dive

- **VNet (10.0.0.0/16):** The isolated private network boundary for this workload. Sized generously (/16) even though only two small subnets exist today — resizing a VNet's address space later is disruptive, so architects over-provision the VNet CIDR even at Level 1.
- **Subnets:** `web-subnet` (10.0.1.0/24) isolates the VM; `bastion-subnet` (10.0.2.0/27) is a **mandatory** Azure Bastion requirement — it must be named exactly `AzureBastionSubnet` and be at least /26.
- **NSG:** Applied at the subnet level on `web-subnet`. Inbound rules: allow 443 from Internet, allow SSH (22) only from the Bastion subnet range, explicit deny-all below default rules. NSGs are stateful — a rule allowing inbound 443 automatically allows the matching outbound response traffic.
- **UDRs (User Defined Routes):** Not needed yet — there's no firewall or NVA to force-tunnel traffic through. This becomes relevant starting Level 3 when Azure Firewall is introduced.
- **DNS:** Using Azure-provided default DNS for now; a custom domain (e.g., shop.contosoboutique.com) will point via a CNAME/A record at the VM's public IP, with the TLS cert from Key Vault terminating HTTPS on Nginx.
- **Peering / VPN / ExpressRoute:** Not applicable — single VNet, no on-prem connectivity requirement at this stage. This changes in Level 6 (Hybrid Architectures).
- **Load Balancer / App Gateway / Azure Firewall:** Not present — single instance, nothing to balance across, and Azure Firewall is overkill for one VM's egress. Introducing these at Level 1 would be over-engineering; an architect should be able to justify *removing* complexity as easily as adding it.

---

## 6. Security Deep Dive

- **Microsoft Entra ID:** The one admin's account lives in Entra ID; no local VM-only accounts are used for portal/CLI access.
- **RBAC:** The admin is assigned **Contributor** scoped to the single Resource Group only — not Owner, and not at the Subscription level. This is the most commonly missed exam point on AZ-104/AZ-500: scope role assignments as narrowly as the task allows.
- **PIM (Privileged Identity Management):** Not yet justified — PIM's value is eligible/just-in-time elevation for multiple admins; with one admin and one resource group, standing Contributor access is an acceptable, documented risk at this scale. PIM becomes mandatory starting Level 3.
- **Managed Identity:** A System-Assigned Managed Identity is enabled on the VM, granted `Key Vault Secrets User` role on the Key Vault — so the app retrieves the DB connection string at runtime with zero embedded credentials.
- **Key Vault:** Access policy uses **Azure RBAC mode** (not the legacy access-policy model) for consistency with the rest of the RBAC strategy.
- **Defender for Cloud:** Free tier enabled for basic security posture scoring; Defender for Servers (paid plan) is **not** yet enabled — deferred to Level 3 once multiple VMs justify the per-resource cost.
- **Sentinel:** Not applicable — Sentinel's value is correlating signals across many resources/sources; a single VM has nothing to correlate against yet.
- **WAF:** Not present — there's no Application Gateway/Front Door yet to attach a WAF policy to. This is the #1 thing added in Level 2 once real internet traffic volume justifies it.
- **DDoS Protection:** Using DDoS **Infrastructure-level protection** (free, automatic, always on for every Azure public IP). DDoS **Network Protection** (paid, ~$3,000/month) is explicitly not justified at this traffic scale — a good interview answer is knowing *why not* to recommend the expensive tier reflexively.

---

## 7. High Availability Design

- **Availability Zones / Sets:** Deliberately **not used** at Level 1 — single VM means no redundancy target exists to distribute across zones. This is an honest limitation we document, not hide.
- **Region Pairs:** App deployed in **East US**, whose paired region is **West US**; relevant for future geo-redundant backup decisions, not active yet.
- **Disaster Recovery:** None today beyond DB backups. RTO/RPO are explicitly defined as a business decision, not a technical afterthought:
  - **RTO (Recovery Time Objective): 8 business hours** — acceptable for a holiday-season side e-commerce channel, not the core business (physical stores still operate).
  - **RPO (Recovery Point Objective): 24 hours** — daily automated SQL backup is sufficient; losing up to a day of orders is a tolerated risk at this revenue scale, explicitly signed off by the owner.
- **Backup Strategy:** Azure SQL Database automated backups (7-day point-in-time restore, included free). VM itself is **not** backed up via Azure Backup at Level 1 — it's treated as disposable/rebuildable from a deployment script (a foreshadow of Infrastructure as Code in Section 10).

---

## 8. Monitoring and Observability

- **Azure Monitor:** Collects VM host metrics (CPU, memory, disk, network) automatically.
- **Log Analytics Workspace:** Central destination for the VM's syslog/auth logs and SQL Database diagnostic logs (query performance, deadlocks, failed connections).
- **Application Insights:** Lightweight SDK added to the Node.js app for request tracing and exception logging — this is the cheapest, highest-value addition a beginner architecture can make and is often skipped by mistake.
- **Alerts:** Two alert rules configured: (1) VM CPU > 80% for 10 minutes → email the admin (early warning before B-series credit exhaustion causes throttling); (2) DTU consumption on SQL DB > 90% → email admin.
- **Dashboards:** A single Azure Dashboard pinned with VM CPU, SQL DTU %, and Application Insights failed-request rate — the three numbers that matter for a one-VM shop.

---

## 9. Cost Optimization

- **Reserved Instances:** Not purchased yet — workload pattern is unvalidated for the first 1-3 months; committing to a 1-year RI before knowing if the VM size is even correct is a classic beginner mistake that locks in the wrong SKU.
- **Savings Plans:** Same reasoning — deferred until Month 3+ once usage is stable, covered properly in Level 7.
- **Auto Scaling:** Not applicable — a single VM can't auto-scale horizontally (no VMSS yet); vertical resize is a manual, planned activity during low-traffic windows.
- **Storage Tiers:** Backup/log data in Storage Account uses **Cool** tier (infrequent access) rather than Hot, since backups are write-once/rarely-read.
- **Cost Analysis:** Owner reviews Cost Management + Billing weekly during the first quarter to validate actual spend against the $150-250/month target band; budget alert configured at $200.

**Estimated Monthly Cost (steady state):**
| Resource | SKU | Est. Cost/mo |
|---|---|---|
| VM | Standard_B2s | ~$60 |
| Managed Disk | 64GB Premium SSD | ~$10 |
| Azure SQL DB | Standard S1 | ~$30 |
| Storage Account | LRS, Cool | ~$3 |
| Bastion | Basic SKU | ~$140 (see Sec. 4 caveat) |
| Key Vault | Standard | ~$1 |
| Log Analytics | Pay-as-you-go, low volume | ~$5 |
| **Total** | | **~$249/mo** |

Note this exceeds the $150 target *because of Bastion* — flagged honestly in Section 13 as a real trade-off decision the owner needs to make, not hidden.

---

## 10. Terraform Version — Resource Hierarchy (no code yet)

Dependency order matters in Terraform — here's the hierarchy before we write a single `.tf` file:

```
azurerm_resource_group (root — everything below depends on this)
│
├── azurerm_virtual_network
│     └── azurerm_subnet (web-subnet)
│     └── azurerm_subnet (AzureBastionSubnet)
│
├── azurerm_network_security_group
│     └── azurerm_subnet_network_security_group_association (binds NSG -> web-subnet)
│
├── azurerm_public_ip (for the VM)
├── azurerm_public_ip (for Bastion — separate, required)
│
├── azurerm_network_interface
│     (depends on: subnet, public_ip)
│
├── azurerm_linux_virtual_machine
│     (depends on: network_interface)
│     └── identity block (System-Assigned Managed Identity — created implicitly, no separate resource)
│
├── azurerm_bastion_host
│     (depends on: AzureBastionSubnet, bastion public_ip)
│
├── azurerm_mssql_server
│     └── azurerm_mssql_database
│     └── azurerm_mssql_virtual_network_rule (depends on: web-subnet, mssql_server)
│
├── azurerm_storage_account
│
├── azurerm_key_vault
│     └── azurerm_key_vault_access_policy / azurerm_role_assignment
│           (depends on: linux_virtual_machine's identity principal_id)
│
└── azurerm_log_analytics_workspace
      └── azurerm_monitor_diagnostic_setting (attached to VM, SQL server)
```

**Key dependency insight for the exam and real life:** the Key Vault role assignment *cannot* be created until the VM's Managed Identity exists — Terraform handles this automatically via implicit dependency (`principal_id` reference), but if you ever see a `depends_on` used here in someone's code, ask why — it usually means they didn't realize the identity reference already creates an implicit dependency, a common code-review flag.

Full `.tf` code will be provided once you've drawn this dependency graph yourself from memory — that's the actual skill, not copy-pasting HCL.

---

## 11. Interview Preparation

**AZ-104 (Administrator) — 10 Questions**
1. What's the difference between an NSG applied to a subnet vs. a NIC?
2. How do you move a VM to a different Resource Group, and what are the limitations?
3. Explain the difference between Azure Disk types: Standard HDD, Standard SSD, Premium SSD.
4. What happens to a VM's public IP if you stop (not deallocate) it?
5. How do you grant a user access to only one Resource Group without giving Subscription-level access?
6. What's the difference between a Basic and Standard Load Balancer? (Relevant once we add one.)
7. How does Azure Backup differ for a VM vs. Azure SQL Database?
8. What's the difference between resizing a VM and changing its disk performance tier?
9. How do management groups, subscriptions, resource groups, and resources relate in the hierarchy?
10. What's the purpose of an Activity Log vs. a Diagnostic Setting?

**AZ-500 (Security) — 10 Questions**
1. Walk through how a VM's Managed Identity retrieves a secret from Key Vault — what's actually happening under the hood (Azure AD token flow)?
2. What's the difference between Key Vault Access Policies and Azure RBAC for Key Vault?
3. Why is Azure Bastion considered more secure than a public RDP/SSH endpoint?
4. What's Just-In-Time (JIT) VM access in Defender for Cloud, and how does it differ from Bastion?
5. Explain Azure SQL Database's VNet service endpoint/firewall rule vs Private Link — trade-offs?
6. What does Defender for Cloud's Secure Score actually measure?
7. How would you detect that someone disabled an NSG rule outside of change control?
8. Why would you choose System-Assigned vs User-Assigned Managed Identity?
9. What's the security risk of standing (non-PIM) Contributor access, even scoped to one RG?
10. How does Azure encrypt data at rest for a Managed Disk by default, and can you bring your own key?

**AZ-700 (Networking) — 10 Questions**
1. Why must the Bastion subnet be named exactly `AzureBastionSubnet`, and what's the minimum size?
2. Explain stateful vs stateless NSG rule evaluation.
3. What's the difference between a Service Endpoint and a Private Endpoint for Azure SQL Database?
4. How does Azure assign DNS names by default within a VNet, and when would you need a Private DNS Zone?
5. What's the order of NSG rule evaluation — does priority number matter, and which wins?
6. Why is a /16 VNet a reasonable default even for a single-VM workload?
7. What's the difference between a Basic Public IP SKU and a Standard SKU, and which works with Standard Load Balancer?
8. When would User Defined Routes (UDRs) become necessary in this architecture as it grows?
9. How do NSG flow logs help troubleshoot a "connection timeout" issue?
10. What's the difference between Azure-provided DNS and a custom Private DNS Zone for name resolution between the VM and a future second VM?

**AZ-305 (Architect) — 10 Questions**
1. Justify why App Service was *not* chosen here, and at what point would you recommend migrating off the VM?
2. How would you redesign this for 99.99% SLA instead of 99.9%? What specifically changes?
3. What's your argument for/against Bastion's recurring cost at this small a scale — what's the alternative and its risk?
4. How would this architecture need to change to support a second region for DR?
5. At what concurrent-user threshold does a single Standard_B2s VM become the bottleneck, and what's your next architectural move?
6. Why is DTU-based SQL purchasing appropriate now but possibly wrong in 12 months?
7. How would you introduce CI/CD without redesigning the network?
8. What's your case for keeping payment processing entirely outside this architecture (PCI scope reduction)?
9. If the business suddenly needed multi-region failover, what's the minimum viable change vs. the ideal change?
10. How do you justify Bastion's cost to a cost-conscious owner who asks "can we just open port 22 instead"?

---

## 12. Hands-On Lab

**Goal:** Build this exact architecture in your own subscription.

**Estimated Cost to run the lab for ~4 hours then tear down:** under $2 (Bastion billed hourly is the main cost driver — deploy it last, test, then delete the Bastion resource specifically if you want to keep the rest running cheaply).

**Step-by-step:**
1. Create Resource Group `rg-contoso-boutique-lab` in `East US`.
2. Create VNet `vnet-contoso` (10.0.0.0/16) with subnet `web-subnet` (10.0.1.0/24).
3. Add subnet `AzureBastionSubnet` (10.0.2.0/27) — exact name required.
4. Create NSG `nsg-web`, add inbound rule allow 443 from Internet (priority 100), allow 22 from 10.0.2.0/27 only (priority 110); associate to `web-subnet`.
5. Deploy Ubuntu 22.04 VM (Standard_B2s), attach to `web-subnet`, enable System-Assigned Managed Identity during creation.
6. Deploy Azure Bastion (Basic SKU) into `AzureBastionSubnet`.
7. Connect via Bastion in the portal — confirm you reach the VM **without** any public SSH exposure (verify by trying — and failing — to SSH directly from your laptop to the VM's public IP).
8. Create Azure SQL Server + Database (Standard S1); under Networking, set "Public network access: Disabled" and add a VNet rule for `web-subnet`.
9. Create Key Vault; grant the VM's Managed Identity the `Key Vault Secrets User` role via Access Control (IAM) — not the legacy access policy blade.
10. Store a dummy secret (e.g., a fake connection string) in Key Vault; from inside the VM (via Bastion), use Azure CLI with the Managed Identity (`az login --identity`) to retrieve it — confirm no credentials were ever typed.
11. Create a Log Analytics Workspace; enable Diagnostic Settings on the VM and SQL Database pointing to it.
12. Create one Metric Alert: VM CPU > 80% for 10 min → action group emails you.
13. **Tear down:** delete the Resource Group entirely when done to stop all billing.

**Common Mistakes**
- Forgetting the Bastion subnet must be named exactly `AzureBastionSubnet` (case-sensitive) — deployment will fail otherwise.
- Leaving SQL Server's public network access enabled "just to test," then forgetting to disable it — this is the #1 real-world misconfiguration that leads to data breaches.
- Using the legacy Key Vault access policy model instead of RBAC — fine for the lab, but inconsistent with the RBAC strategy taught here; pick one model and justify it.
- Forgetting to actually attempt the direct SSH connection in step 7 — without trying and failing, you haven't actually validated the security control.

**Validation Steps**
- [ ] Direct SSH to the VM's public IP from your laptop times out / is refused.
- [ ] Bastion connection succeeds.
- [ ] `az login --identity` from inside the VM successfully retrieves the Key Vault secret with zero stored credentials.
- [ ] Attempting to connect to the SQL Database from a tool outside the VNet (e.g., your laptop's SSMS) fails.
- [ ] CPU alert fires when you artificially load the VM (e.g., `stress --cpu 2 --timeout 600`).

---

## 13. Architecture Review

**Why this design is good**
It is honest about scale — every component maps directly to a stated requirement, nothing is gold-plated. It establishes three habits that carry through every future level: Managed Identity instead of stored credentials, network isolation of the database, and no public management ports. Getting these right at Level 1 means you never have to "bolt on" security later — it's load-bearing from day one.

**Limitations**
- Single point of failure: the VM. Any OS-level issue, patching reboot, or B-series CPU throttling event takes the site down with no failover.
- No WAF — a 200-SKU catalog site is still a target for bot scraping and basic injection attempts; Nginx alone provides no application-layer protection.
- Bastion's ~$140/month is disproportionate to the rest of the ~$110/month workload — a real architect would raise this trade-off explicitly with the business rather than silently accepting it.
- Manual deployment (SSH in, `git pull`, restart) is not repeatable or auditable — one typo during a holiday-season hotfix could cause an outage with no rollback plan.

**How a senior architect would improve it (preview of Level 2)**
Move the app to Azure App Service (removes OS patching burden, built-in deployment slots for safe rollouts), add Application Gateway with WAF in front, and introduce a second App Service instance for actual redundancy — at which point a Load Balancer/App Gateway choice becomes a real conversation, not a hypothetical.

**Scaling from 100 users to 1,000,000 users**
- **100–1K users:** This Level 1 design holds, possibly resize VM to B4ms.
- **1K–50K users:** Level 2 — App Service + deployment slots, Application Gateway + WAF, move to vCore SQL with read replicas.
- **50K–500K users:** Level 3/4 — VMSS or AKS, Azure Front Door for global edge caching, multi-region active-passive, Cosmos DB or SQL geo-replication.
- **500K–1M+ users:** Level 10 — Front Door + multi-region active-active, AKS with HPA, Cosmos DB multi-region writes, Azure Cache for Redis, full CI/CD with progressive rollout, Chaos Engineering validation of failover.

The architecture pattern doesn't change randomly between levels — each jump is a direct response to a new bottleneck or new requirement, which is exactly how you should defend design decisions in an AZ-305 exam scenario or a real design review.

---

## 14. Portfolio and GitHub Documentation

### A. GitHub Repository Structure

```
contoso-boutique-azure-lab/
│
├── architecture-diagram/
│   └── level1-diagram.png   (export the ASCII diagram or redraw in draw.io)
│
├── screenshots/
│   └── (see list below)
│
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md  (explains the resource hierarchy from Section 10)
│
├── scripts/
│   └── deploy-app.sh   (the manual git-pull/restart script used at this level)
│
├── docs/
│   ├── requirements.md
│   ├── security-decisions.md
│   └── cost-analysis.md
│
└── README.md
```

### B. Screenshots Required
- Resource Group overview (all resources listed)
- Virtual Network + subnet layout
- NSG rules (inbound list, showing the deny-all)
- VM overview page (showing Managed Identity enabled)
- Azure Bastion connection screen mid-session
- SQL Database Networking blade (showing "Public access: Disabled")
- Key Vault Access Control (IAM) role assignment for the VM identity
- Successful `az login --identity` + secret retrieval in terminal (via Bastion)
- Log Analytics Workspace with VM heartbeat data flowing in
- Metric Alert rule configuration and a fired alert email/notification
- Cost Analysis chart for the Resource Group
- Failed SSH attempt from your local laptop (proof of the security control working)

### C. README.md Template

```markdown
# Contoso Boutique — Azure Level 1 Architecture

## Project Overview
A secure, minimal-footprint single-VM web application architecture built
for a small retail business launching an e-commerce site under budget
and time constraints.

## Business Use Case
[Insert Section 1 content]

## Architecture Diagram
[Insert diagram image]

## Azure Services Used
- Azure VM (Linux), Azure SQL Database, Key Vault, Azure Bastion,
  Storage Account, Log Analytics, Azure Monitor

## Security Design
- Managed Identity for zero-credential secret retrieval
- Database isolated to VNet only, no public access
- No public management ports (Bastion-only access)
- RBAC scoped to Resource Group, not Subscription

## Network Design
- Single VNet (10.0.0.0/16), two subnets
- NSG with explicit allow-list, deny-all default

## Deployment Steps
[Link to terraform/README.md]

## Validation Steps
[Insert Section 12 validation checklist]

## Challenges Faced
- [Document real issues you hit during your own lab build]

## Lessons Learned
- [Your own reflection — this is what interviewers actually want to hear]

## Cost Estimation
[Insert Section 9 cost table]

## Future Improvements
- Migrate to App Service, add WAF, introduce CI/CD (see Level 2)
```

### D. Interview Story (STAR Method)

**Situation:** "A small retail client needed a customer-facing e-commerce site live within six weeks, with no existing cloud infrastructure or in-house DevOps team, and a strict sub-$300/month operating budget."

**Task:** "I was responsible for designing a production-safe architecture that balanced security fundamentals against the realistic constraints of a one-person IT team and an unproven traffic pattern."

**Action:** "I deployed a single Linux VM behind Azure Bastion to eliminate public management-port exposure, used a System-Assigned Managed Identity so the application never stored a database credential anywhere, and isolated Azure SQL Database to VNet-only access. I documented every component against a specific business requirement, explicitly excluding services like WAF and Availability Zones that the scale didn't yet justify — including a documented trade-off discussion on Bastion's cost relative to the rest of the environment."

**Result:** "The site launched on schedule, passed a basic security review with zero embedded credentials found in code, and I left the client with a clear, requirement-mapped path for when to introduce App Service, WAF, and high availability as their traffic grows — rather than over-engineering a $250/month workload into a $2,000/month one on day one."

### E. Resume Bullet Points
- Designed and deployed a secure single-tier Azure web architecture using Azure VM, Azure SQL Database, Key Vault, and Azure Bastion, eliminating all public management-port exposure.
- Implemented credential-less application authentication using Azure Managed Identity and Key Vault RBAC, removing hardcoded secrets from the codebase.
- Architected network isolation for a PaaS database using VNet service endpoints, restricting connectivity to application-tier resources only.
- Established baseline monitoring and alerting using Azure Monitor and Log Analytics, including proactive CPU and database utilization alerting.
- Produced a cost-justified architecture decision document mapping every Azure resource to an explicit business or security requirement.

### F. Production Improvements (what a real enterprise adds before go-live)
- CI/CD pipeline (Azure DevOps or GitHub Actions) replacing manual SSH deployment.
- Application Gateway + WAF in front of the VM, even before moving to App Service.
- Azure Backup for the VM itself (not just the database).
- A second admin account with PIM-based eligible access instead of one standing Contributor account (single-person bus-factor risk).
- Formal incident response runbook — what happens at 2 AM if the VM CPU alert fires and the one admin is asleep?

### G. Portfolio Score

| Lens | Score /10 | Why |
|---|---|---|
| AZ-104 | 8/10 | Strong coverage of VM, NSG, Bastion, RBAC fundamentals; missing Backup Vault and VM availability concepts (deliberately, for this level). |
| AZ-500 | 7/10 | Solid on Managed Identity, Key Vault RBAC, network isolation; no Defender for Cloud paid tier or PIM usage yet — appropriate for scope, but caps the score until added. |
| AZ-700 | 6/10 | Good subnet/NSG/Bastion fundamentals; no Load Balancer, App Gateway, UDRs, or DNS zones yet — those arrive Level 2-3. |
| AZ-305 | 7/10 | Strong requirement-to-design traceability and explicit trade-off documentation (the thing AZ-305 scenarios actually test); limited by intentionally small scope. |
| Resume Value | 6/10 | Good foundational story, but "single VM" projects alone won't differentiate you — value compounds as you stack Levels 2-10 into one portfolio narrative. |
| Interview Value | 8/10 | The STAR story above is strong specifically *because* it explains trade-offs and what was deliberately excluded — that's senior-level thinking, not checkbox deployment. |

**How to improve the score:** Complete Level 2 next and link this repo to it as "Phase 1 of 2" — interviewers respond strongly to seeing an evolving architecture with documented reasoning at each stage, more than to any single isolated project.

---

**Next lesson (Day 2, Level 1 continued):** We'll take this same business and add a second VM behind a Standard Load Balancer — your first real availability design decision, and the first time NSG rules need to account for backend pool health probes.
