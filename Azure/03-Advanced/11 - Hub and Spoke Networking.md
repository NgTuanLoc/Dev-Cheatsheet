---
tags: [azure, advanced, networking, architecture]
aliases: [Hub-Spoke, Hub and Spoke, vWAN]
level: Advanced
---

# Hub and Spoke Networking

> **One-liner**: **Hub-and-spoke** is the canonical Azure network topology — one **hub VNet** owns shared services (firewall, VPN/ExpressRoute, DNS), each **spoke VNet** is a workload, and **VNet peering** connects them while a **firewall** in the hub forces all cross-spoke traffic through inspection.

---

## Quick Reference

| Component | Purpose |
| --------- | ------- |
| **Hub VNet** | Shared services: firewall, VPN/ExpressRoute gateway, Bastion, DNS |
| **Spoke VNet** | One per workload / app team |
| **VNet Peering** | Layer-3 link between two VNets (non-transitive by default) |
| **Azure Firewall** | Stateful L4-L7 firewall in the hub; FQDN + threat intel rules |
| **User-Defined Route (UDR)** | Forces 0.0.0.0/0 through firewall (next-hop = AzFW) |
| **Private DNS Zone** | Resolution for private endpoints, linked to the hub VNet |
| **ExpressRoute / VPN GW** | On-prem connectivity — single gateway in hub serves all spokes |
| **Azure Virtual WAN** | Managed hub-of-hubs across regions (alternative to manual hubs) |

| vNet vs vWAN | Choose |
| ------------ | ------ |
| **Manual hub (single region)** | Small footprint, full control |
| **Manual hubs (multi-region with global peering)** | More control; more wiring |
| **Virtual WAN (vWAN)** | Multi-region, secure hub, managed transitive routing |

---

## Core Concept

A flat VNet for everything works until you have two app teams; then traffic isolation, shared services, and on-prem connectivity get political. Hub-and-spoke solves this by giving every workload its own VNet (the **spoke**) and centralizing what they all need in a single **hub**.

**Peering is the connector.** Two peerings (A→B and B→A) make traffic flow. Crucially, peering is **non-transitive**: spoke-A peered to hub, hub peered to spoke-B does **not** let A reach B unless you add **UDRs** that send traffic through the hub firewall.

**Azure Firewall in the hub** becomes the single chokepoint. UDRs on each spoke subnet force `0.0.0.0/0 → 10.0.0.4` (the firewall private IP). Firewall rules then decide what spokes can talk to.

**Shared services live once.** ExpressRoute gateway, VPN gateway, Azure Bastion, Private DNS Zones, log forwarders — all in the hub. Spokes never duplicate these; they peer to the hub and inherit reachability.

**vWAN is the managed alternative**: Microsoft creates and runs the hubs, gives you transitive routing across regions, and integrates Azure Firewall as a "secure hub." Higher minimum cost, less wiring.

---

## Diagram

```mermaid
graph TB
    subgraph On-prem
      DC[Datacenter]
    end
    subgraph Hub VNet 10.0.0.0/16
      AFW[Azure Firewall]
      ER[ExpressRoute Gateway]
      DNS[Private DNS Zones]
      Bast[Azure Bastion]
    end
    subgraph Spoke A 10.10.0.0/16
      AppA[App workload]
      PEa[Private endpoints]
    end
    subgraph Spoke B 10.20.0.0/16
      AppB[App workload]
    end
    subgraph Spoke Identity 10.30.0.0/16
      ADDS[AD DS]
    end
    DC <-->|MSEE| ER
    Hub:::hub
    Hub <-.peering.-> Spoke_A
    Hub <-.peering.-> Spoke_B
    Hub <-.peering.-> Spoke_Identity
    AppA -.UDR 0.0.0.0/0.-> AFW
    AppB -.UDR 0.0.0.0/0.-> AFW
    AppA -.cross-spoke.-> AFW -.allowed.-> AppB
    PEa -.private DNS.-> DNS
```

---

## Syntax & API

### Build a hub + two spokes

```bash
RG=rg-net-prod
LOC=eastus

az group create -n $RG -l $LOC

# Hub VNet
az network vnet create -g $RG -n vnet-hub --address-prefix 10.0.0.0/16 \
  --subnet-name AzureFirewallSubnet --subnet-prefix 10.0.1.0/26
az network vnet subnet create -g $RG --vnet-name vnet-hub -n GatewaySubnet --address-prefix 10.0.2.0/27
az network vnet subnet create -g $RG --vnet-name vnet-hub -n AzureBastionSubnet --address-prefix 10.0.3.0/26

# Spoke A
az network vnet create -g $RG -n vnet-spoke-a --address-prefix 10.10.0.0/16 \
  --subnet-name app --subnet-prefix 10.10.1.0/24

# Spoke B
az network vnet create -g $RG -n vnet-spoke-b --address-prefix 10.20.0.0/16 \
  --subnet-name app --subnet-prefix 10.20.1.0/24

# Peerings (both directions, allow gateway transit from hub)
az network vnet peering create -g $RG -n hub-to-a --vnet-name vnet-hub --remote-vnet vnet-spoke-a --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
az network vnet peering create -g $RG -n a-to-hub --vnet-name vnet-spoke-a --remote-vnet vnet-hub --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways
az network vnet peering create -g $RG -n hub-to-b --vnet-name vnet-hub --remote-vnet vnet-spoke-b --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
az network vnet peering create -g $RG -n b-to-hub --vnet-name vnet-spoke-b --remote-vnet vnet-hub --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways
```

### Deploy Azure Firewall in the hub

```bash
az network public-ip create -g $RG -n pip-afw --sku Standard --allocation-method Static
az network firewall create -g $RG -n afw-hub --sku AZFW_VNet --tier Standard
az network firewall ip-config create -g $RG -f afw-hub -n ipc1 \
  --vnet-name vnet-hub --public-ip-address pip-afw

AFW_PIP=$(az network firewall show -g $RG -n afw-hub --query 'ipConfigurations[0].privateIPAddress' -o tsv)
echo "Firewall private IP: $AFW_PIP"
```

### Force spoke egress through firewall

```bash
az network route-table create -g $RG -n rt-spoke-a
az network route-table route create -g $RG --route-table-name rt-spoke-a \
  -n default-via-afw --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance \
  --next-hop-ip-address $AFW_PIP

az network vnet subnet update -g $RG --vnet-name vnet-spoke-a -n app --route-table rt-spoke-a
```

### Allow only HTTPS to specific FQDNs

```bash
az network firewall application-rule create -g $RG -f afw-hub \
  --collection-name app-allow --priority 100 --action Allow \
  --name web-allowlist --source-addresses 10.10.1.0/24 \
  --protocols Https=443 \
  --target-fqdns api.stripe.com login.microsoftonline.com management.azure.com
```

### Link a Private DNS Zone to the hub (so spokes resolve PaaS PE)

```bash
az network private-dns zone create -g $RG -n privatelink.blob.core.windows.net
az network private-dns link vnet create -g $RG -n hub-link \
  --zone-name privatelink.blob.core.windows.net --virtual-network vnet-hub --registration-enabled false
# Spokes resolve via hub since they peer to it (DNS forwarder/private resolver pattern)
```

---

## Common Patterns

- **One hub per region**, spokes per workload. Cross-region: global VNet peering or vWAN.
- **Azure Firewall Premium** for TLS inspection + IDS/IPS where compliance requires it. Standard for most.
- **Azure Private DNS Resolver** in the hub if spokes need conditional forwarding to on-prem AD or if Private DNS Zone resolution requires custom forwarders.
- **Single subscription for connectivity** (per ALZ pattern); spokes live in workload subs and peer in.
- **NSGs on every subnet** as a deny-default safety net even when firewall handles egress.
- **Service Tags > IPs** in firewall rules — Microsoft updates `Storage.eastus` automatically.
- **vWAN for global enterprises** with 5+ regions; manual hubs for ≤3 regions or where you need fine control over routing.

---

## Gotchas & Tips

- **Peering is non-transitive.** Spoke-A → Hub → Spoke-B doesn't work without UDRs forcing traffic through the hub firewall. This trips up everyone the first time.
- **`AzureFirewallSubnet`, `GatewaySubnet`, `AzureBastionSubnet` are reserved names** with minimum size requirements. Don't get creative.
- **Azure Firewall costs ~$1.25/hr** plus $0.016/GB. Not free. For low-traffic dev hubs, an NVA or no firewall is cheaper.
- **Global VNet peering doesn't allow gateway transit** — each region's hub needs its own gateway, or use vWAN.
- **Private endpoints don't honor NSGs by default.** Recent updates allow it, but ensure `enable-network-security-group-rules` is set on the subnet.
- **Asymmetric routing kills sessions.** A response that goes a different path than the request will be dropped by the stateful firewall. Always design symmetric routes.
- **UDR + 0.0.0.0/0 to firewall = no internet from that subnet** unless the firewall allows it. Test before deploying everywhere.
- **`AllowGatewayTransit` + `UseRemoteGateways`** must both be set for spokes to reach on-prem via hub's ExpressRoute/VPN gateway.
- **vWAN secured hubs use Azure Firewall Manager**, not standalone AzFW config — different rule structure (firewall policies).
- **Address-space overlap is forbidden** between peered VNets. Plan your CIDR allocation upfront; renumbering is brutal.
- **Charges per peering connection per GB** in each direction. Cross-region is more expensive than intra-region.

---

## See Also

- [[17 - VNet and Subnets]]
- [[18 - Application Gateway and Load Balancer]]
- [[12 - Private Endpoints and Zero Trust]]
- [[02 - Landing Zones]]
