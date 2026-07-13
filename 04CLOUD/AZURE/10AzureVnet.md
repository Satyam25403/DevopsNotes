# Azure Virtual Network (VNet)
## (analogous to AWS VPC)

Azure Virtual Network is the foundational networking layer in Azure. It provides private, isolated networking for your Azure resources — VMs, AKS clusters, App Services, databases, and more. Like a VPC, you define the IP space, subnets, and routing; Azure handles the underlying physical network.

---

## Core Concepts

| Concept | Description | AWS Equivalent |
|---------|-------------|----------------|
| **VNet** | Isolated private network with a CIDR range | VPC |
| **Subnet** | Subdivision of the VNet's address space | Subnet |
| **Network Security Group (NSG)** | Stateful firewall with allow/deny rules | Security Group |
| **Route Table** | Custom routes for subnet traffic | Route Table |
| **VNet Peering** | Connect two VNets privately | VPC Peering |
| **VNet Gateway** | VPN or ExpressRoute connectivity | Virtual Private Gateway |
| **Private Endpoint** | Private IP for a PaaS service inside your VNet | VPC Endpoint (Interface) |
| **Service Endpoint** | Route PaaS traffic through Azure backbone | VPC Endpoint (Gateway) |
| **Azure Bastion** | Browser-based SSH/RDP without public IPs | EC2 Instance Connect / Session Manager |
| **NAT Gateway** | Managed outbound internet for private subnets | NAT Gateway |

---

## Creating a VNet

```bash
# Create a VNet with an address space
az network vnet create \
  --resource-group myRG \
  --name myVNet \
  --address-prefix 10.0.0.0/16 \
  --location eastus

# Add subnets
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
```

---

## Network Security Groups (NSGs)

NSGs filter traffic with priority-ordered rules. Lower number = higher priority. Rules 65000–65500 are default Azure rules (allow VNet/LB, deny internet).

```bash
# Create an NSG
az network nsg create \
  --resource-group myRG \
  --name webNSG

# Allow inbound HTTP and HTTPS
az network nsg rule create \
  --resource-group myRG \
  --nsg-name webNSG \
  --name AllowHTTPS \
  --priority 100 \
  --direction Inbound \
  --protocol Tcp \
  --source-address-prefixes "*" \
  --destination-port-ranges 443 \
  --access Allow

# Allow SSH only from specific IP
az network nsg rule create \
  --resource-group myRG \
  --nsg-name webNSG \
  --name AllowSSH \
  --priority 110 \
  --direction Inbound \
  --protocol Tcp \
  --source-address-prefixes "203.0.113.10" \
  --destination-port-ranges 22 \
  --access Allow

# Associate NSG with a subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name webSubnet \
  --network-security-group webNSG

# List rules
az network nsg rule list --resource-group myRG --nsg-name webNSG --output table
```

---

## VNet Peering (cross-VNet connectivity)

```bash
# Peer VNet-A → VNet-B
az network vnet peering create \
  --resource-group myRG \
  --name VNetA-to-VNetB \
  --vnet-name VNetA \
  --remote-vnet VNetB \
  --allow-vnet-access

# Peer VNet-B → VNet-A (peering is not automatic in both directions)
az network vnet peering create \
  --resource-group myRG \
  --name VNetB-to-VNetA \
  --vnet-name VNetB \
  --remote-vnet VNetA \
  --allow-vnet-access
```

> Unlike VPC Peering, Azure VNet peering requires two peering objects — one in each direction.

---

## Private Endpoints (bring PaaS services inside your VNet)

Private Endpoints give Azure PaaS services (Storage, Key Vault, SQL, Cosmos DB, etc.) a private IP inside your subnet. Traffic never leaves the Azure backbone.

```bash
# Create a private endpoint for a storage account
az network private-endpoint create \
  --resource-group myRG \
  --name myStoragePrivateEndpoint \
  --vnet-name myVNet \
  --subnet dataSubnet \
  --private-connection-resource-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage \
  --group-id blob \
  --connection-name myStorageConnection

# Create a private DNS zone for blob storage
az network private-dns zone create \
  --resource-group myRG \
  --name "privatelink.blob.core.windows.net"

# Link DNS zone to the VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "privatelink.blob.core.windows.net" \
  --name myDNSLink \
  --virtual-network myVNet \
  --registration-enabled false

# Create DNS record for the private endpoint
az network private-endpoint dns-zone-group create \
  --resource-group myRG \
  --endpoint-name myStoragePrivateEndpoint \
  --name myZoneGroup \
  --private-dns-zone "privatelink.blob.core.windows.net" \
  --zone-name blob
```

---

## NAT Gateway (managed outbound internet for private subnets)

```bash
# Create a public IP for the NAT Gateway
az network public-ip create \
  --resource-group myRG \
  --name myNATPublicIP \
  --sku Standard

# Create the NAT Gateway
az network nat gateway create \
  --resource-group myRG \
  --name myNATGateway \
  --public-ip-addresses myNATPublicIP \
  --idle-timeout 10

# Associate with a subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name appSubnet \
  --nat-gateway myNATGateway
```

---

## Azure Bastion (secure SSH/RDP without public IPs)

```bash
# Create Bastion (requires a dedicated AzureBastionSubnet with /27 or larger)
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name AzureBastionSubnet \
  --address-prefix 10.0.255.0/27

az network public-ip create \
  --resource-group myRG \
  --name myBastionIP \
  --sku Standard

az network bastion create \
  --resource-group myRG \
  --name myBastion \
  --public-ip-address myBastionIP \
  --vnet-name myVNet \
  --location eastus
```

Once deployed, SSH/RDP into VMs directly from the Azure Portal browser — no public IPs on the VMs needed.

---

## Load Balancers

### Azure Load Balancer (L4 — analogous to AWS NLB)

```bash
az network lb create \
  --resource-group myRG \
  --name myLoadBalancer \
  --sku Standard \
  --frontend-ip-name myFrontendIP \
  --backend-pool-name myBackendPool
```

### Application Gateway (L7 — analogous to AWS ALB)

```bash
az network application-gateway create \
  --resource-group myRG \
  --name myAppGateway \
  --vnet-name myVNet \
  --subnet appGatewaySubnet \
  --capacity 2 \
  --sku WAF_v2 \
  --http-settings-cookie-based-affinity Disabled \
  --frontend-port 443 \
  --http-settings-port 80 \
  --public-ip-address myAppGatewayIP
```

---

## Key Differences from AWS VPC

| Feature | AWS VPC | Azure VNet |
|---------|---------|-----------|
| Firewall | Security Groups (stateful) + NACLs (stateless) | NSGs (stateful, applied to NIC or subnet) |
| Stateless rules | NACLs | No direct equivalent (NSGs are stateful) |
| Cross-VNet | VPC Peering | VNet Peering (bidirectional objects required) |
| Private PaaS access | VPC Endpoints | Private Endpoints + Private DNS |
| Outbound NAT | NAT Gateway | NAT Gateway |
| No-IP SSH | EC2 Instance Connect / SSM Session Manager | Azure Bastion |
| L7 load balancer | ALB | Application Gateway |
| L4 load balancer | NLB | Azure Load Balancer (Standard) |
| DNS | Route 53 Private Zones | Azure Private DNS Zones |
| Internet Gateway | Implicit per VPC | Not needed — subnets with public IPs route automatically |