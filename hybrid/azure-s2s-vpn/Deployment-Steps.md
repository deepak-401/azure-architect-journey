# Deployment Steps — Azure Site-to-Site VPN

Step-by-step guide to establish a Site-to-Site (S2S) VPN connection between an on-premises network and Azure.

## Prerequisites

- Active Azure subscription
- An on-premises VPN device (physical or virtual firewall/router) that supports IKEv1/IKEv2 IPsec — e.g. pfSense, Cisco ASR, FortiGate, MikroTik, or Windows Server RRAS
- A **static public IP address** on the on-premises VPN device (required — Azure needs a fixed endpoint to peer with)
- Admin access to configure the on-prem device
- Knowledge of your on-premises network's private address range(s)

---

## Step 1 — Create the Virtual Network (VNet)

1. In the Azure Portal, search **Virtual networks → + Create**
2. Resource group: create or select one, e.g. `rg-s2s-vpn-demo`
3. Name: `vnet-s2s-demo`
4. Region: choose a region close to your on-prem location
5. IPv4 address space: `10.0.0.0/16`
6. Add a subnet for your workloads, e.g.:
   - Name: `snet-workload`
   - Address range: `10.0.1.0/24`
7. **Review + Create → Create**

> Make sure this address space does **not overlap** with your on-premises network range.

## Step 2 — Create the GatewaySubnet

1. Open the Virtual Network → **Subnets → + Gateway subnet**
2. Azure names this `GatewaySubnet` automatically (required — do not rename)
3. Address range: `10.0.255.0/27` (a `/27` or larger is recommended)
4. **Save**

## Step 3 — Deploy the Azure VPN Gateway

1. Search **Virtual network gateways → + Create**
2. Name: `vgw-s2s-demo`
3. Region: same as the VNet
4. Gateway type: **VPN**
5. VPN type: **Route-based**
6. SKU: `VpnGw1` (minimum recommended for production S2S; `Basic` works for lab/testing only)
7. Virtual network: select `vnet-s2s-demo`
8. Public IP address: **Create new** → name it `pip-vgw-s2s-demo`
9. **Review + Create → Create**

> Provisioning typically takes 30–45 minutes. Use this time to gather your on-prem device's public IP and address range for the next step.

## Step 4 — Create the Local Network Gateway

This resource represents your **on-premises side** from Azure's perspective.

1. Search **Local network gateways → + Create**
2. Name: `lng-onprem-site`
3. Endpoint: **IP address**
4. IP address: your on-premises device's **static public IP**
5. Address space: your on-premises private network range(s), e.g. `192.168.0.0/16`
6. Region: same as your VNet
7. **Review + Create → Create**

## Step 5 — Create the VPN Connection (Configure Shared Key)

1. Open the **Virtual Network Gateway** (`vgw-s2s-demo`) → **Connections → + Add**
2. Name: `conn-onprem-to-azure`
3. Connection type: **Site-to-site (IPsec)**
4. Virtual network gateway: `vgw-s2s-demo`
5. Local network gateway: `lng-onprem-site`
6. Shared key (PSK): enter a strong, random pre-shared key — **note it down**, you'll need the exact same value on the on-prem device
7. IKE Protocol: **IKEv2**
8. **OK / Create**

## Step 6 — Configure the On-Premises VPN Device

On your on-prem firewall/router, create a matching Site-to-Site IPsec tunnel:

1. **Remote peer (Azure side)**: the public IP of `pip-vgw-s2s-demo`
2. **Local network**: your on-prem address range (must match what you entered in Step 4)
3. **Remote network**: your Azure VNet range (`10.0.0.0/16`)
4. **Pre-shared key**: exactly the same value entered in Step 5
5. **IKE version**: IKEv2
6. **Phase 1 (IKE) settings**: match Azure's defaults — typically AES256, SHA256, DH Group 14/2048-bit
7. **Phase 2 (IPsec) settings**: AES256, SHA256, PFS Group 14 (adjust based on your device's supported policies)
8. Save and enable the tunnel

> Exact steps vary by vendor. Refer to your device's documentation or Microsoft's [About VPN devices](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpn-devices) page for vendor-specific configuration guides and validated device tables.

## Step 7 — Verify the IPsec Tunnel

1. In Azure Portal, open the **Connection** resource (`conn-onprem-to-azure`)
2. Check **Status** — it should show **Connected** once both sides negotiate successfully
3. Review **Ingress/Egress bytes** to confirm traffic is flowing

Validate end-to-end connectivity:

```bash
# From an Azure VM in the VNet, ping an on-prem host
ping 192.168.x.x

# From an on-prem machine, ping an Azure VM's private IP
ping 10.0.1.x
```

Capture the following as validation evidence for `Screenshots/`:
- VPN Gateway overview showing **Succeeded** provisioning state
- Connection resource showing status **Connected**
- On-prem device's tunnel status page showing the tunnel **up**
- A successful ping/traceroute between on-prem and Azure

---

## Troubleshooting

| Issue | Likely Cause |
|---|---|
| Connection status stuck on "Connecting" | Shared key mismatch, or on-prem device not initiating the tunnel |
| Tunnel up but no traffic passes | NSG blocking traffic, or missing route on either side |
| Tunnel drops intermittently | Phase 1/Phase 2 lifetime mismatch between Azure and on-prem device |
| Can't create Local Network Gateway | On-prem public IP not static, or already in use by another connection |
| Overlapping address space error | On-prem and Azure VNet ranges overlap — redesign addressing |

## Cleanup

To avoid ongoing charges, delete the resource group when done:

```bash
az group delete --name rg-s2s-vpn-demo --yes --no-wait
```
