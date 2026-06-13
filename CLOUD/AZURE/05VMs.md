# Azure Virtual Machines (VMs)
## (analogous to Amazon EC2)

Azure VMs provide scalable, resizable virtual servers in the cloud. Like EC2, they're the backbone of Azure compute — used for web apps, ML workloads, databases, and anything requiring a full OS.

---

## Core Concepts

### Virtual Machine
A virtualized server running on Azure infrastructure. You choose the OS (Linux/Windows), CPU, memory, storage, and networking. Billed per second (or per minute on some SKUs).

### VM Image : VMI (analogous to AMI)
A template used to create VMs. Contains the OS and pre-installed software.
- **Azure Marketplace images**: Ubuntu, Windows Server, RHEL, CentOS, etc.
- **Custom images**: Created from an existing VM (captured via Azure Compute Gallery)
- **Azure Compute Gallery** (formerly Shared Image Gallery): Stores and replicates custom images across regions.

### VM Sizes (analogous to EC2 instance types)

| Series | Optimized For | Example |
|--------|--------------|---------|
| B-series | Burstable general-purpose | B1s (free tier equivalent) |
| D/Dv5-series | Balanced general-purpose | D2s_v5 |
| F-series | Compute-intensive | F4s_v2 |
| E-series | Memory-intensive | E8s_v5 |
| N-series | GPU / ML training | NC6s_v3 |
| L-series | Storage-optimized | L8s_v3 |
| H-series | High-performance computing | H16r |

### SSH Keys
Used for Linux VM authentication. Azure stores the public key; you keep the private key.

```bash
# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -f ~/.ssh/azure_key

# SSH into VM with private key(400 permission)
ssh -i <path_of_priv_key> username@ip
```

### Network Security Groups (NSGs) (analogous to EC2 Security Groups)
Virtual firewalls controlling inbound/outbound traffic. Rules specify source/destination IP, port, protocol, and allow/deny.

---

## Creating a VM

### Via CLI
```bash
az vm create \
  --resource-group myRG \
  --name myVM \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard

# SSH into the VM
az vm show -d --resource-group myRG --name myVM --query publicIps -o tsv
ssh azureuser@<public-ip>
```

### Open a port (add NSG rule)
```bash
az vm open-port --resource-group myRG --name myVM --port 80
az vm open-port --resource-group myRG --name myVM --port 443
```

---

## VM Lifecycle

```bash
az vm start --resource-group myRG --name myVM
az vm stop --resource-group myRG --name myVM        # deallocated = no charge
az vm restart --resource-group myRG --name myVM
az vm deallocate --resource-group myRG --name myVM  # fully deallocated
az vm delete --resource-group myRG --name myVM
az vm list --output table
```

> **Stopped vs Deallocated**: `az vm stop` from inside the OS keeps the VM allocated (you're still charged). `az vm deallocate` releases the compute — no charge. This is different from EC2 where "stop" always deallocates.

---

## Storage: Managed Disks (analogous to EBS)

| Disk Type | Description | EC2 Equivalent |
|-----------|-------------|----------------|
| **Standard HDD** | Low-cost, low throughput | gp2 (HDD) |
| **Standard SSD** | General-purpose | gp2 |
| **Premium SSD** | High IOPS, low latency | io1/io2 |
| **Ultra Disk** | Mission-critical, configurable IOPS | io2 Block Express |

```bash
# Add a data disk to an existing VM
az vm disk attach \
  --resource-group myRG \
  --vm-name myVM \
  --name myDataDisk \
  --new \
  --size-gb 128 \
  --sku Premium_LRS

# Detach a disk
az vm disk detach --resource-group myRG --vm-name myVM --name myDataDisk
```

---

## VM Extensions (analogous to EC2 User Data / Systems Manager)

Extensions run scripts or install software at provisioning time:

```bash
# Run a custom script at VM creation
az vm create \
  --resource-group myRG \
  --name myVM \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init.yaml

# Apply a script extension post-creation
az vm extension set \
  --resource-group myRG \
  --vm-name myVM \
  --name customScript \
  --publisher Microsoft.Azure.Extensions \
  --settings '{"commandToExecute":"apt-get install -y nginx"}'
```

---

## VM Scale Sets (analogous to EC2 Auto Scaling Groups)

Automatically create and manage a group of identical VMs with auto-scaling:

```bash
az vmss create \
  --resource-group myRG \
  --name myScaleSet \
  --image Ubuntu2204 \
  --upgrade-policy-mode automatic \
  --admin-username azureuser \
  --generate-ssh-keys \
  --instance-count 2 \
  --vm-sku Standard_B1s

# Scale manually
az vmss scale --resource-group myRG --name myScaleSet --new-capacity 5

# Configure autoscale
az monitor autoscale create \
  --resource-group myRG \
  --resource myScaleSet \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-config \
  --min-count 2 \
  --max-count 10 \
  --count 2
```

---

## Pricing Models (analogous to EC2 pricing)

| Model | Description | EC2 Equivalent |
|-------|-------------|----------------|
| **Pay-as-you-go** | Per-second billing | On-Demand |
| **Reserved Instances** | 1 or 3 year commitment, up to 72% discount | Reserved Instances |
| **Spot VMs** | Use spare capacity, up to 90% discount, can be evicted | Spot Instances |
| **Azure Hybrid Benefit** | Use existing Windows/Linux licenses | AWS Dedicated Hosts |

---

## Availability Options (analogous to EC2 AZ placement)

| Option | Protection From | EC2 Equivalent |
|--------|----------------|----------------|
| **Availability Zones** | Datacenter failure | Multi-AZ deployment |
| **Availability Sets** | Rack/hardware failure within datacenter | Placement Groups |
| **VM Scale Sets (across zones)** | Zone + hardware failure | ASG + AZ |

```bash
# Create VM in a specific Availability Zone
az vm create \
  --resource-group myRG \
  --name myVM \
  --image Ubuntu2204 \
  --zone 1 \
  --admin-username azureuser \
  --generate-ssh-keys
```

---

## Networking

Each VM gets:
- **vNIC** (Virtual Network Interface Card)
- **Private IP** from the Virtual Network (VNet) subnet
- Optional **Public IP**
- **NSG** (Network Security Group) attached to vNIC or subnet

```bash
# Show VM network details
az vm show \
  --resource-group myRG \
  --name myVM \
  --show-details \
  --query "{PublicIP:publicIps, PrivateIP:privateIps, FQDN:fqdns}"
```

---

## Key Differences from EC2

| Feature | AWS EC2 | Azure VM |
|---------|---------|---------|
| Image template | AMI | Azure Compute Gallery image |
| Disk | EBS Volumes | Managed Disks |
| Firewall | Security Groups | Network Security Groups (NSGs) |
| Auto-scaling | Auto Scaling Groups | VM Scale Sets |
| User data/scripts | EC2 User Data | Custom Data (cloud-init) + VM Extensions |
| Instance metadata | Instance Metadata Service | Azure Instance Metadata Service (IMDS) |
| Stopped state billing | No charge when stopped | No charge only when **deallocated** |
| SSH keys | Key Pairs | SSH public keys stored in VM |