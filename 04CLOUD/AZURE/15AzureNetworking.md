# Azure Networking
## (analogous to AWS VPC, Route 53, CloudFront & Global Accelerator)

Azure networking spans four layers: **Virtual Networks** (private isolated networks), **DNS** (name resolution), **CDN & Front Door** (global edge delivery), and **Traffic Manager** (DNS-based global routing). This document covers all of them.

---

## Part 1: Azure Virtual Network (VNet)
## (analogous to AWS VPC)

Azure Virtual Network is the foundational networking primitive. Every resource that needs private connectivity — VMs, AKS clusters, App Services, databases — lives inside a VNet. You define the IP space, subnets, and routing; Azure handles the physical underlay.

---

### Core Concepts

| Concept | Description | AWS Equivalent |
|---------|-------------|----------------|
| **VNet** | Isolated private network with a CIDR range | VPC |
| **Subnet** | Subdivision of the VNet's address space | Subnet |
| **Network Security Group (NSG)** | Stateful firewall rules on a subnet or NIC | Security Group |
| **Route Table** | Custom routes overriding Azure's defaults | Route Table |
| **VNet Peering** | Connect two VNets privately, same or cross-region | VPC Peering |
| **VNet Gateway** | VPN or ExpressRoute connectivity to on-prem | Virtual Private Gateway |
| **Private Endpoint** | Private IP for a PaaS service inside your VNet | VPC Endpoint (Interface) |
| **Service Endpoint** | Route PaaS traffic through Azure backbone (no private IP) | VPC Endpoint (Gateway) |
| **Azure Bastion** | Browser-based SSH/RDP without public IPs on VMs | EC2 Instance Connect / SSM Session Manager |
| **NAT Gateway** | Managed outbound internet for private subnets | NAT Gateway |
| **NIC (Network Interface Card)** | Virtual network adapter attached to a VM | Elastic Network Interface (ENI) |
| **Public IP** | A static or dynamic public IPv4/IPv6 address | Elastic IP (EIP) |

---

### How It All Fits Together

```
Subscription
  └── Resource Group
        └── VNet  (e.g., 10.0.0.0/16)
              ├── Subnet: webSubnet   (10.0.1.0/24)  ← NSG attached
              ├── Subnet: appSubnet   (10.0.2.0/24)  ← NSG attached, NAT Gateway
              ├── Subnet: dataSubnet  (10.0.3.0/24)  ← NSG attached, Private Endpoints
              └── Subnet: AzureBastionSubnet (10.0.255.0/27)

VM → NIC → Private IP (from subnet) + optional Public IP
PaaS (Storage, SQL, Key Vault) → Private Endpoint → Private IP in dataSubnet
```

> Unlike AWS, Azure VNets do **not** have an explicit Internet Gateway resource. Any subnet with a resource assigned a Public IP automatically has outbound internet access unless a Route Table or Firewall blocks it.

---

### Creating a VNet and Subnets

```bash
# Create a VNet
az network vnet create \
  --resource-group myRG \
  --name myVNet \
  --address-prefix 10.0.0.0/16 \
  --location eastus

# Create subnets (each carves out a slice of the VNet CIDR)
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name webSubnet \
  --address-prefix 10.0.1.0/24

az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name appSubnet \
  --address-prefix 10.0.2.0/24

az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name dataSubnet \
  --address-prefix 10.0.3.0/24

# List subnets
az network vnet subnet list \
  --resource-group myRG \
  --vnet-name myVNet \
  --output table

# Show VNet details (address space, subnets, peerings)
az network vnet show \
  --resource-group myRG \
  --name myVNet
```

> Azure reserves **5 IP addresses** in every subnet: .0 (network), .1 (default gateway), .2–.3 (Azure DNS), .255 (broadcast). A /24 gives you 251 usable addresses.

---

### Public IPs

```bash
# Create a static Standard public IP (required for Load Balancer, Bastion, NAT Gateway)
az network public-ip create \
  --resource-group myRG \
  --name myPublicIP \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3          # zone-redundant

# Create a basic dynamic public IP (legacy — avoid for new resources)
az network public-ip create \
  --resource-group myRG \
  --name myPublicIP \
  --sku Basic \
  --allocation-method Dynamic

# List public IPs
az network public-ip list --resource-group myRG --output table

# Show assigned IP address
az network public-ip show \
  --resource-group myRG \
  --name myPublicIP \
  --query ipAddress \
  --output tsv
```

> **Standard vs Basic**: Always use Standard SKU for new deployments. Standard is zone-redundant, supports Availability Zones, and is required for most modern services. Basic is being retired.

---

### Network Interfaces (NICs)

A NIC is the virtual adapter connecting a VM to a subnet. VMs can have multiple NICs (for multi-homed scenarios).

```bash
# Create a NIC (usually done implicitly during VM creation)
az network nic create \
  --resource-group myRG \
  --name myNIC \
  --vnet-name myVNet \
  --subnet appSubnet \
  --network-security-group appNSG \
  --public-ip-address ""        # no public IP — private only

# List NICs
az network nic list --resource-group myRG --output table

# Show effective routes on a NIC (useful for debugging routing)
az network nic show-effective-route-table \
  --resource-group myRG \
  --name myNIC \
  --output table

# Show effective NSG rules on a NIC (what rules are actually applied)
az network nic list-effective-nsg \
  --resource-group myRG \
  --name myNIC
```

---

### Network Security Groups (NSGs)

NSGs are stateful Layer 3/4 firewalls. Rules are priority-ordered — lower number = higher priority. They can be attached to a **subnet** (applies to all resources in the subnet) or to an individual **NIC** (applies only to that VM). Both can be applied simultaneously; Azure evaluates both.

Default rules you cannot delete (priority 65000–65500):
- `AllowVnetInBound` — allow all traffic between resources in the same VNet
- `AllowAzureLoadBalancerInBound` — allow health probes from Azure Load Balancer
- `DenyAllInBound` — deny everything else inbound
- `AllowVnetOutBound` / `AllowInternetOutBound` — allow outbound to VNet and internet
- `DenyAllOutBound` — deny everything else outbound

```bash
# Create an NSG
az network nsg create \
  --resource-group myRG \
  --name webNSG

# Allow inbound HTTPS from anywhere
az network nsg rule create \
  --resource-group myRG \
  --nsg-name webNSG \
  --name AllowHTTPS \
  --priority 100 \
  --direction Inbound \
  --protocol Tcp \
  --source-address-prefixes "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges 443 \
  --access Allow

# Allow inbound HTTP (redirect to HTTPS in app)
az network nsg rule create \
  --resource-group myRG \
  --nsg-name webNSG \
  --name AllowHTTP \
  --priority 110 \
  --direction Inbound \
  --protocol Tcp \
  --source-address-prefixes "*" \
  --destination-port-ranges 80 \
  --access Allow

# Allow SSH only from a jump box IP
az network nsg rule create \
  --resource-group myRG \
  --nsg-name webNSG \
  --name AllowSSH \
  --priority 120 \
  --direction Inbound \
  --protocol Tcp \
  --source-address-prefixes "10.0.0.4" \
  --destination-port-ranges 22 \
  --access Allow

# Deny all other inbound (explicit — actually redundant due to default deny, but makes intent clear)
az network nsg rule create \
  --resource-group myRG \
  --nsg-name webNSG \
  --name DenyAllInbound \
  --priority 4096 \
  --direction Inbound \
  --protocol "*" \
  --source-address-prefixes "*" \
  --destination-port-ranges "*" \
  --access Deny

# Use Service Tags instead of IPs (Azure-managed groups of IP ranges)
az network nsg rule create \
  --resource-group myRG \
  --nsg-name appNSG \
  --name AllowAzureMonitor \
  --priority 200 \
  --direction Outbound \
  --protocol Tcp \
  --source-address-prefixes "VirtualNetwork" \
  --destination-address-prefixes "AzureMonitor" \
  --destination-port-ranges 443 \
  --access Allow

# Associate NSG with a subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name webSubnet \
  --network-security-group webNSG

# Associate NSG with a NIC instead
az network nic update \
  --resource-group myRG \
  --name myNIC \
  --network-security-group appNSG

# List rules in an NSG
az network nsg rule list \
  --resource-group myRG \
  --nsg-name webNSG \
  --output table
```

**Service Tags** — use these instead of IP ranges for Azure services:

| Tag | Covers |
|-----|--------|
| `VirtualNetwork` | All addresses in the VNet + peered VNets |
| `AzureLoadBalancer` | Health probe source IPs |
| `Internet` | Any IP outside the VNet |
| `AzureMonitor` | Azure Monitor / Log Analytics IPs |
| `Storage` | Azure Storage service IPs |
| `Sql` | Azure SQL and Synapse IPs |
| `AppService` | Azure App Service outbound IPs |

---

### Route Tables (User-Defined Routes)

Azure automatically routes traffic within a VNet, to the internet, and to peered VNets. You add a custom Route Table to override this — most commonly to force all outbound traffic through an Azure Firewall or NVA (Network Virtual Appliance).

```bash
# Create a route table
az network route-table create \
  --resource-group myRG \
  --name appRouteTable \
  --disable-bgp-route-propagation false

# Add a route: send all internet traffic to the firewall
az network route-table route create \
  --resource-group myRG \
  --route-table-name appRouteTable \
  --name forceInternetToFirewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.255.4   # private IP of the Azure Firewall

# Add a route: send specific CIDR directly (bypass firewall for internal VNet)
az network route-table route create \
  --resource-group myRG \
  --route-table-name appRouteTable \
  --name localVNet \
  --address-prefix 10.0.0.0/16 \
  --next-hop-type VnetLocal

# Associate route table with a subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name appSubnet \
  --route-table appRouteTable

# List routes
az network route-table route list \
  --resource-group myRG \
  --route-table-name appRouteTable \
  --output table
```

**Next-hop types:**

| Type | Sends traffic to |
|------|-----------------|
| `VnetLocal` | Within the VNet (default) |
| `Internet` | Azure's internet edge |
| `VirtualAppliance` | A specific private IP (firewall, NVA) |
| `VirtualNetworkGateway` | VPN or ExpressRoute gateway |
| `None` | Drop the traffic (blackhole) |

---

### VNet Peering

Peering connects two VNets privately over the Azure backbone — no internet, no encryption overhead, low latency. Works within a region (VNet peering) and across regions (global VNet peering).

```bash
# Peer VNet-A → VNet-B
az network vnet peering create \
  --resource-group myRG \
  --name VNetA-to-VNetB \
  --vnet-name VNetA \
  --remote-vnet /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/VNetB \
  --allow-vnet-access \
  --allow-forwarded-traffic    # allow traffic forwarded from other networks

# Peer VNet-B → VNet-A (must create both directions — not automatic)
az network vnet peering create \
  --resource-group myRG \
  --name VNetB-to-VNetA \
  --vnet-name VNetB \
  --remote-vnet /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/VNetA \
  --allow-vnet-access \
  --allow-forwarded-traffic

# List peerings on a VNet
az network vnet peering list \
  --resource-group myRG \
  --vnet-name VNetA \
  --output table
```

> VNet peering is **non-transitive** by default. If VNet-A peers with VNet-B and VNet-B peers with VNet-C, traffic from VNet-A cannot reach VNet-C. To enable transitive routing, use **Azure Virtual WAN** or a hub VNet with **Azure Firewall** and forwarded traffic enabled.

---

### Private Endpoints

A Private Endpoint gives a PaaS service (Storage, Key Vault, SQL, Cosmos DB, Service Bus, etc.) a private IP inside your subnet. Traffic stays on the Azure backbone — it never traverses the public internet.

```bash
# Create a private endpoint for a storage account (blob)
az network private-endpoint create \
  --resource-group myRG \
  --name myStoragePrivateEndpoint \
  --vnet-name myVNet \
  --subnet dataSubnet \
  --private-connection-resource-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage \
  --group-id blob \
  --connection-name myStoragePEConnection

# Each PaaS service has a different --group-id:
# Storage: blob, file, queue, table, dfs
# Key Vault: vault
# SQL: sqlServer
# Cosmos DB: Sql, MongoDB, Cassandra, Gremlin, Table
# Service Bus: namespace
# App Configuration: configurationStores

# Create the matching private DNS zone (so hostnames resolve to the private IP)
az network private-dns zone create \
  --resource-group myRG \
  --name "privatelink.blob.core.windows.net"

# Link the DNS zone to the VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "privatelink.blob.core.windows.net" \
  --name myDNSLink \
  --virtual-network myVNet \
  --registration-enabled false

# Auto-register the private endpoint in the DNS zone
az network private-endpoint dns-zone-group create \
  --resource-group myRG \
  --endpoint-name myStoragePrivateEndpoint \
  --name myZoneGroup \
  --private-dns-zone "privatelink.blob.core.windows.net" \
  --zone-name blob
```

**Private DNS zone names per service:**

| Service | Private DNS Zone |
|---------|-----------------|
| Blob Storage | `privatelink.blob.core.windows.net` |
| File Storage | `privatelink.file.core.windows.net` |
| Key Vault | `privatelink.vaultcore.azure.net` |
| Azure SQL | `privatelink.database.windows.net` |
| Cosmos DB (SQL) | `privatelink.documents.azure.com` |
| Service Bus | `privatelink.servicebus.windows.net` |
| App Config | `privatelink.azconfig.io` |
| Container Registry | `privatelink.azurecr.io` |

---

### Service Endpoints (lightweight alternative to Private Endpoints)

Service Endpoints extend your VNet's identity to a PaaS service over the Azure backbone — but without a private IP. The PaaS service still has a public hostname; you just whitelist the VNet as an allowed source.

```bash
# Enable Service Endpoint for Storage on a subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name appSubnet \
  --service-endpoints Microsoft.Storage Microsoft.KeyVault Microsoft.Sql

# Then restrict the storage account to only allow traffic from that subnet
az storage account network-rule add \
  --resource-group myRG \
  --account-name mystorage \
  --vnet-name myVNet \
  --subnet appSubnet

az storage account update \
  --resource-group myRG \
  --name mystorage \
  --default-action Deny   # deny everything not explicitly allowed
```

> **Private Endpoint vs Service Endpoint**: Private Endpoints give the service a private IP and work from on-premises (via VPN/ExpressRoute). Service Endpoints are simpler but only work from within the VNet and don't block public access by themselves. Prefer Private Endpoints for production.

---

### NAT Gateway

Provides a stable, managed outbound internet connection for private subnets (no public IPs on individual VMs required). All VMs in the subnet share the NAT Gateway's public IP(s) for outbound connections.

```bash
# Create a Standard public IP for the NAT Gateway
az network public-ip create \
  --resource-group myRG \
  --name myNATPublicIP \
  --sku Standard \
  --allocation-method Static

# Optionally create a public IP prefix (a block of IPs — useful for whitelisting)
az network public-ip prefix create \
  --resource-group myRG \
  --name myNATIPPrefix \
  --length 31            # 2 public IPs

# Create the NAT Gateway
az network nat gateway create \
  --resource-group myRG \
  --name myNATGateway \
  --public-ip-addresses myNATPublicIP \
  --idle-timeout 10      # minutes before idle TCP connections are closed

# Associate with a subnet (replaces the default outbound route for that subnet)
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name appSubnet \
  --nat-gateway myNATGateway
```

---

### Azure Bastion (SSH/RDP without public IPs)

Azure Bastion deploys into a dedicated subnet and provides browser-based SSH and RDP to VMs in the VNet — no public IP needed on the VMs, no VPN client required.

```bash
# Create the required AzureBastionSubnet (/27 minimum)
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name AzureBastionSubnet \
  --address-prefix 10.0.255.0/27

# Create a Standard public IP for Bastion
az network public-ip create \
  --resource-group myRG \
  --name myBastionIP \
  --sku Standard \
  --allocation-method Static

# Deploy Bastion (takes ~5 minutes)
az network bastion create \
  --resource-group myRG \
  --name myBastion \
  --public-ip-address myBastionIP \
  --vnet-name myVNet \
  --location eastus \
  --sku Standard      # Standard enables tunneling and native client support

# SSH to a VM via Bastion (no public IP on the VM)
az network bastion ssh \
  --resource-group myRG \
  --name myBastion \
  --target-resource-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --auth-type ssh-key \
  --username azureuser \
  --ssh-key ~/.ssh/azure_key
```

---

### VNet Gateway (VPN / ExpressRoute)

For connecting your on-premises network to Azure.

```bash
# Create a GatewaySubnet (required — must be named exactly this)
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name GatewaySubnet \
  --address-prefix 10.0.254.0/27

# Create a public IP for the gateway
az network public-ip create \
  --resource-group myRG \
  --name myGatewayIP \
  --sku Standard \
  --allocation-method Static

# Create a VPN Gateway (takes 20–45 minutes)
az network vnet-gateway create \
  --resource-group myRG \
  --name myVPNGateway \
  --location eastus \
  --vnet myVNet \
  --public-ip-address myGatewayIP \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1 \
  --generation Generation1

# Create a connection to on-prem (Site-to-Site VPN)
az network vpn-connection create \
  --resource-group myRG \
  --name toOnPrem \
  --vnet-gateway1 myVPNGateway \
  --local-gateway2 myLocalNetworkGateway \
  --shared-key "MyPreSharedKey123!"
```

---

### Load Balancers

#### Azure Load Balancer (L4 — analogous to AWS NLB)

Distributes TCP/UDP traffic across VMs in a backend pool. No awareness of HTTP — pure IP/port balancing.

```bash
# Create a Standard public load balancer
az network lb create \
  --resource-group myRG \
  --name myLB \
  --sku Standard \
  --frontend-ip-name myFrontendIP \
  --public-ip-address myPublicIP \
  --backend-pool-name myBackendPool

# Add a health probe (check port 80 every 15s)
az network lb probe create \
  --resource-group myRG \
  --lb-name myLB \
  --name httpProbe \
  --protocol Http \
  --port 80 \
  --path /health \
  --interval 15 \
  --threshold 2

# Add a load balancing rule
az network lb rule create \
  --resource-group myRG \
  --lb-name myLB \
  --name httpRule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name myFrontendIP \
  --backend-pool-name myBackendPool \
  --probe-name httpProbe

# Add a VM NIC to the backend pool
az network nic ip-config address-pool add \
  --resource-group myRG \
  --nic-name myVM-NIC \
  --ip-config-name ipconfig1 \
  --lb-name myLB \
  --address-pool myBackendPool
```

#### Application Gateway (L7 — analogous to AWS ALB)

HTTP/HTTPS aware load balancer. Supports URL-based routing, SSL termination, session affinity, WAF, and header rewriting.

```bash
# Create a dedicated subnet for App Gateway (must be its own subnet)
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name appGatewaySubnet \
  --address-prefix 10.0.10.0/24

az network public-ip create \
  --resource-group myRG \
  --name myAppGatewayIP \
  --sku Standard \
  --allocation-method Static

# Create an Application Gateway with WAF v2
az network application-gateway create \
  --resource-group myRG \
  --name myAppGateway \
  --location eastus \
  --vnet-name myVNet \
  --subnet appGatewaySubnet \
  --public-ip-address myAppGatewayIP \
  --sku WAF_v2 \
  --capacity 2 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --frontend-port 443 \
  --servers 10.0.2.4 10.0.2.5   # backend IPs

# Add URL path routing (route /api/* to different backend)
az network application-gateway url-path-map create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name myPathMap \
  --paths "/api/*" \
  --address-pool apiBackendPool \
  --http-settings apiHttpSettings \
  --default-address-pool webBackendPool \
  --default-http-settings webHttpSettings
```

---

## Part 2: Azure DNS
## (analogous to AWS Route 53)

Azure DNS hosts your DNS zones on Azure's global anycast network. Integrates natively with Azure resources and supports public and private zones.

---

### Public DNS Zones

```bash
# Create a public DNS zone
az network dns zone create \
  --resource-group myRG \
  --name mydomain.com

# A record
az network dns record-set a add-record \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name www \
  --ipv4-address 203.0.113.10

# CNAME record
az network dns record-set cname set-record \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name api \
  --cname my-app.azurewebsites.net

# MX record
az network dns record-set mx add-record \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name "@" \
  --exchange mail.mydomain.com \
  --preference 10

# TXT record (SPF, domain verification)
az network dns record-set txt add-record \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name "@" \
  --value "v=spf1 include:sendgrid.net ~all"

# List all records
az network dns record-set list \
  --resource-group myRG \
  --zone-name mydomain.com \
  --output table

# Get name servers (point your registrar to these 4)
az network dns zone show \
  --resource-group myRG \
  --name mydomain.com \
  --query nameServers \
  --output table
```

---

### Alias Records (analogous to Route 53 Alias)

Alias records resolve directly to Azure resource IPs — they follow dynamic IP changes and don't bill for queries to Azure targets.

```bash
# Alias A record pointing to a Standard Public IP
az network dns record-set a create \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name "@" \
  --target-resource /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Network/publicIPAddresses/myPublicIP

# Alias pointing to a Traffic Manager profile
az network dns record-set a create \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name "@" \
  --target-resource /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Network/trafficManagerProfiles/myTMProfile
```

---

### Private DNS Zones (internal VNet resolution)

```bash
# Create a private DNS zone
az network private-dns zone create \
  --resource-group myRG \
  --name internal.mydomain.com

# Link to a VNet (enables resolution from VMs in the VNet)
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name internal.mydomain.com \
  --name myVNetLink \
  --virtual-network myVNet \
  --registration-enabled true    # auto-registers VM hostnames in the zone

# Manually add an internal record
az network private-dns record-set a add-record \
  --resource-group myRG \
  --zone-name internal.mydomain.com \
  --record-set-name db \
  --ipv4-address 10.0.3.10

# List records in the private zone
az network private-dns record-set list \
  --resource-group myRG \
  --zone-name internal.mydomain.com \
  --output table
```

---

## Part 3: Traffic Manager
## (analogous to Route 53 routing policies — DNS-based global load balancing)

Traffic Manager uses DNS to route users to endpoints across regions. It is not a proxy — it returns an IP and the client connects directly to the endpoint.

| Routing Method | Behavior | Route 53 Equivalent |
|----------------|----------|---------------------|
| **Priority** | Primary/failover — use secondary when primary is unhealthy | Failover routing |
| **Weighted** | Split traffic by weight across endpoints | Weighted routing |
| **Performance** | Route to lowest-latency endpoint from user's location | Latency routing |
| **Geographic** | Route by user's country or region | Geolocation routing |
| **Multivalue** | Return multiple healthy endpoints | Multivalue answer |
| **Subnet** | Route based on the user's IP subnet | — |

```bash
# Create a Traffic Manager profile
az network traffic-manager profile create \
  --resource-group myRG \
  --name myTMProfile \
  --routing-method Performance \
  --unique-dns-name my-global-app \   # → my-global-app.trafficmanager.net
  --ttl 30 \
  --protocol HTTPS \
  --port 443 \
  --path /health

# Add an Azure endpoint (App Service, Public IP, etc.)
az network traffic-manager endpoint create \
  --resource-group myRG \
  --profile-name myTMProfile \
  --name eastus-endpoint \
  --type azureEndpoints \
  --target-resource-id /subscriptions/<sub-id>/resourceGroups/myRG-eastus/providers/Microsoft.Web/sites/my-app-eastus \
  --endpoint-status Enabled

az network traffic-manager endpoint create \
  --resource-group myRG \
  --profile-name myTMProfile \
  --name westeurope-endpoint \
  --type azureEndpoints \
  --target-resource-id /subscriptions/<sub-id>/resourceGroups/myRG-westeurope/providers/Microsoft.Web/sites/my-app-westeurope \
  --endpoint-status Enabled

# Add an external endpoint (any internet-facing hostname)
az network traffic-manager endpoint create \
  --resource-group myRG \
  --profile-name myTMProfile \
  --name external-endpoint \
  --type externalEndpoints \
  --target other-cloud-app.example.com \
  --endpoint-location "Central India" \
  --endpoint-status Enabled

# Check endpoint health status
az network traffic-manager endpoint show \
  --resource-group myRG \
  --profile-name myTMProfile \
  --type azureEndpoints \
  --name eastus-endpoint \
  --query endpointMonitorStatus
```

---

## Part 4: Azure Front Door
## (analogous to AWS CloudFront + Global Accelerator + WAF)

Azure Front Door is the recommended global entry point for web applications. It operates at Layer 7 (HTTP/HTTPS) and combines CDN caching, global anycast load balancing, TLS termination, and WAF in a single service. Unlike Traffic Manager (DNS-based), Front Door is a true proxy — it terminates connections at the nearest edge PoP and then forwards to your origin.

> Front Door Standard: CDN + global load balancing. Front Door Premium: adds WAF managed rule sets + Private Link to origins.

```bash
# Create a Front Door profile
az afd profile create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --sku Standard_AzureFrontDoor

# Create an endpoint (public-facing hostname)
az afd endpoint create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name my-app-endpoint \
  --enabled-state Enabled
# → my-app-endpoint.z01.azurefd.net

# Create an origin group (pool of backends with health probing)
az afd origin-group create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --probe-request-type GET \
  --probe-protocol Https \
  --probe-interval-in-seconds 30 \
  --probe-path /health \
  --sample-size 4 \
  --successful-samples-required 3

# Add origins (backends — App Service, AKS, VMs, etc.)
az afd origin create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --origin-name eastus-origin \
  --host-name my-app-eastus.azurewebsites.net \
  --origin-host-header my-app-eastus.azurewebsites.net \
  --https-port 443 \
  --priority 1 \
  --weight 1000 \
  --enabled-state Enabled

az afd origin create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --origin-name westeurope-origin \
  --host-name my-app-westeurope.azurewebsites.net \
  --origin-host-header my-app-westeurope.azurewebsites.net \
  --https-port 443 \
  --priority 2 \
  --weight 1000 \
  --enabled-state Enabled

# Create a route (connect endpoint → origin group)
az afd route create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name my-app-endpoint \
  --route-name myRoute \
  --origin-group myOriginGroup \
  --supported-protocols Https \
  --https-redirect Enabled \
  --forwarding-protocol HttpsOnly \
  --link-to-default-domain Enabled \
  --patterns-to-match "/*"
```

---

### Custom Domain + Managed TLS

```bash
az afd custom-domain create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --custom-domain-name myCustomDomain \
  --host-name www.mydomain.com \
  --certificate-type ManagedCertificate    # Azure provisions and auto-renews

az afd route update \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name my-app-endpoint \
  --route-name myRoute \
  --custom-domains myCustomDomain
```

---

### WAF Policy

```bash
# Create a WAF policy
az network front-door waf-policy create \
  --resource-group myRG \
  --name myWAFPolicy \
  --sku Standard_AzureFrontDoor \
  --mode Prevention

# Block a specific IP range
az network front-door waf-policy rule create \
  --resource-group myRG \
  --policy-name myWAFPolicy \
  --name BlockBadIP \
  --priority 100 \
  --rule-type MatchRule \
  --action Block \
  --match-condition \
    matchVariable=RemoteAddr \
    operator=IPMatch \
    values="192.0.2.0/24"

# Rate limit rule (Premium only for managed rules; custom rate limit on Standard)
az network front-door waf-policy rule create \
  --resource-group myRG \
  --policy-name myWAFPolicy \
  --name RateLimitRule \
  --priority 200 \
  --rule-type RateLimitRule \
  --action Block \
  --rate-limit-threshold 1000 \
  --rate-limit-duration-in-minutes 1 \
  --match-condition matchVariable=RemoteAddr operator=IPMatch values="0.0.0.0/0"

# Attach WAF policy to Front Door
az afd security-policy create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --security-policy-name mySecurityPolicy \
  --waf-policy /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Network/frontdoorwebapplicationfirewallpolicies/myWAFPolicy \
  --domains /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Cdn/profiles/myFrontDoor/afdEndpoints/my-app-endpoint
```

---

### Caching Rules

```bash
az afd rule-set create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --rule-set-name myCachingRules

# Cache static assets for 1 day
az afd rule create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --rule-set-name myCachingRules \
  --rule-name CacheStaticAssets \
  --order 1 \
  --match-variable RequestUri \
  --operator RegEx \
  --match-values ".*\.(js|css|png|jpg|woff2|svg)$" \
  --action-name RouteConfigurationOverride \
  --cache-duration "1.00:00:00" \
  --query-string-caching-behavior IgnoreQueryString

# Never cache API responses
az afd rule create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --rule-set-name myCachingRules \
  --rule-name NoCacheAPI \
  --order 2 \
  --match-variable RequestUri \
  --operator BeginsWith \
  --match-values "/api/" \
  --action-name RouteConfigurationOverride \
  --enable-caching false
```

---

## When to Use What

| Scenario | Use |
|----------|-----|
| Private networking between Azure resources | **VNet + Subnets** |
| Filter inbound/outbound traffic to a subnet or VM | **NSG** |
| Force all traffic through a firewall | **Route Table + Azure Firewall** |
| Connect two VNets privately | **VNet Peering** |
| Give a PaaS service a private IP in your VNet | **Private Endpoint** |
| SSH/RDP to a VM without a public IP | **Azure Bastion** |
| Managed outbound internet for private VMs | **NAT Gateway** |
| Connect on-prem to Azure over the internet | **VPN Gateway** |
| Connect on-prem to Azure via dedicated fibre | **ExpressRoute Gateway** |
| Public DNS hosting | **Azure DNS (public zone)** |
| Internal DNS within a VNet | **Azure Private DNS Zone** |
| Route users globally by latency/region/health (DNS-based) | **Traffic Manager** |
| Global HTTP/S delivery with CDN + WAF + failover | **Azure Front Door** |

---

## Key Differences from AWS

| Feature | AWS | Azure |
|---------|-----|-------|
| Private network | VPC | VNet |
| Subnet routing | Route Tables + Internet Gateway | Route Tables (internet access is implicit) |
| Stateful firewall | Security Groups (on instance) | NSG (on subnet or NIC) |
| Stateless firewall | NACLs (on subnet) | No equivalent — NSGs are stateful only |
| Cross-VNet connectivity | VPC Peering | VNet Peering (two objects required) |
| Transitive routing | Transit Gateway | Azure Virtual WAN or hub VNet with Firewall |
| Private PaaS access | VPC Endpoint (Interface) | Private Endpoint + Private DNS |
| PaaS access without private IP | VPC Endpoint (Gateway) | Service Endpoint |
| Outbound NAT | NAT Gateway | NAT Gateway |
| Bastion / no-IP SSH | EC2 Instance Connect / SSM | Azure Bastion |
| VPN to on-prem | Site-to-Site VPN (Virtual Private Gateway) | VPN Gateway |
| Dedicated fibre to on-prem | Direct Connect | ExpressRoute |
| L4 load balancer | NLB | Azure Load Balancer (Standard) |
| L7 load balancer | ALB | Application Gateway |
| Public DNS | Route 53 | Azure DNS |
| Private DNS | Route 53 Private Zones | Azure Private DNS Zones |
| DNS-based global routing | Route 53 routing policies | Traffic Manager |
| Global CDN + proxy + WAF | CloudFront + Global Accelerator + WAF | Azure Front Door |
| Managed TLS on CDN | ACM (free, requires CloudFront) | Front Door Managed Certificate (free) |