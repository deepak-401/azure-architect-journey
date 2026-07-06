# Deployment Steps — Azure P2S VPN with Microsoft Entra ID

Step-by-step guide to reproduce this deployment in the Azure Portal.

## Prerequisites

- Active Azure subscription
- Microsoft Entra ID tenant with permission to register an enterprise application
- Azure VPN Client installed on the connecting device ([download](https://learn.microsoft.com/en-us/azure/vpn-gateway/point-to-site-vpn-client-cert-windows))
- Basic familiarity with the Azure Portal

---

## Step 1 — Create a Resource Group

1. In the Azure Portal, search **Resource groups → + Create**
2. Choose your subscription
3. Name: `rg-p2s-vpn-demo`
4. Region: choose a region close to you (e.g. Central India)
5. **Review + Create → Create**

## Step 2 — Create the Virtual Network

1. Search **Virtual networks → + Create**
2. Select the resource group from Step 1
3. Name: `vnet-p2s-demo`
4. Region: same as the resource group
5. IPv4 address space: `10.0.0.0/16`
6. Add a subnet:
   - Name: `snet-workload`
   - Address range: `10.0.1.0/24`
7. **Review + Create → Create**

## Step 3 — Add the GatewaySubnet

1. Open the Virtual Network → **Subnets → + Gateway subnet**
2. Azure names this subnet `GatewaySubnet` automatically (required — do not rename)
3. Address range: `10.0.255.0/27`
4. **Save**

## Step 4 — Create the Virtual Network Gateway

1. Search **Virtual network gateways → + Create**
2. Name: `vgw-p2s-demo`
3. Region: same as the VNet
4. Gateway type: **VPN**
5. VPN type: **Route-based**
6. SKU: `VpnGw1` (minimum SKU that supports P2S with Entra ID)
7. Virtual network: select `vnet-p2s-demo`
8. Public IP address: **Create new** → name it `pip-vgw-p2s-demo`
9. **Review + Create → Create**

> This step typically takes 30–45 minutes to provision — it's a good time to prepare the VM.

## Step 5 — Register the Azure VPN Entra ID Enterprise Application

1. Go to **Microsoft Entra ID → Enterprise applications**
2. Confirm the Azure VPN application is registered (Azure Portal offers to add it automatically when you configure Entra ID auth on the gateway — see Step 6)
3. Note your **Tenant ID** — needed for the P2S configuration

## Step 6 — Configure Point-to-Site with Microsoft Entra ID Authentication

1. Open the Virtual Network Gateway → **Point-to-site configuration**
2. Address pool: `172.16.0.0/24`
3. Tunnel type: **OpenVPN (SSL)**
4. Authentication type: **Microsoft Entra ID**
5. Fill in:
   - **Tenant**: `https://login.microsoftonline.com/<your-tenant-id>`
   - **Audience**: Azure VPN application ID (`41b23e61-6c1e-4545-b367-cd054e0ed4b4` for the public Azure VPN client app)
   - **Issuer**: `https://sts.windows.net/<your-tenant-id>/`
6. **Save**

## Step 7 — Deploy the Ubuntu Virtual Machine

1. Search **Virtual machines → + Create**
2. Resource group: `rg-p2s-vpn-demo`
3. Name: `vm-workload-01`
4. Image: **Ubuntu Server LTS**
5. Size: any B-series (e.g. `Standard_B1s`) for a lab workload
6. Authentication: SSH public key (or password, for a lab)
7. **Networking tab**:
   - Virtual network: `vnet-p2s-demo`
   - Subnet: `snet-workload`
   - Public IP: **None** ← this is the key step that keeps the VM private
8. **Review + Create → Create**

## Step 8 — Configure the Network Security Group

1. Open the NSG attached to `snet-workload` (or create one and associate it)
2. Add an inbound rule:
   - Source: `VirtualNetwork`
   - Destination port: `22` (SSH)
   - Action: **Allow**
3. Ensure there is no rule allowing inbound access from `Internet`

## Step 9 — Connect via Azure VPN Client

1. On the Virtual Network Gateway, go to **Point-to-site configuration → Download VPN client**
2. Import the downloaded profile into the **Azure VPN Client** app
3. Click **Connect** — this triggers a Microsoft Entra ID sign-in prompt
4. Authenticate with your corporate/tenant credentials
5. Once connected, the client shows **Connected** status with an assigned IP from `172.16.0.0/24`

## Step 10 — Validate Private Connectivity

```bash
ssh <username>@10.0.1.x
```

Replace `10.0.1.x` with the VM's private IP address. If the tunnel and authentication succeeded, this connects directly over the private network — no public IP or open Internet-facing port required.

Capture the following as validation evidence for `Screenshots/`:
- Azure VPN Client showing **Connected**
- Terminal showing a successful SSH session
- VM overview blade showing only a private IP
- VPN Gateway overview blade showing Entra ID authentication configured

---

## Troubleshooting

| Issue | Likely Cause |
|---|---|
| VPN Client fails to connect | Client profile out of date — re-download after any P2S config change |
| Sign-in prompt doesn't appear | Tenant/Audience/Issuer values incorrect in Point-to-site config |
| SSH times out | NSG rule missing, or VM not in the expected subnet |
| Connected but can't reach VM | Check effective routes on the VM's NIC; confirm GatewaySubnet routing |

## Cleanup

To avoid ongoing charges, delete the resource group when done:

```bash
az group delete --name rg-p2s-vpn-demo --yes --no-wait
```
