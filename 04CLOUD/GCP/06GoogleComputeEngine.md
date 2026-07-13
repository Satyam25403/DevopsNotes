# Google Compute Engine (GCE)

Google Compute Engine provides scalable, resizable virtual machines (VMs) in the cloud. It's the backbone of GCP compute — used for everything from hosting web apps to running ML training jobs, and the GCP equivalent of AWS EC2.

---

## Core Concepts

### Instances
A virtual machine running in GCP. You choose the OS (Linux/Windows), machine type (CPU + memory), storage, and networking. Instances are billed per second (minimum 1 minute) with sustained use discounts applied automatically.

### Machine Images vs Snapshots
- **Public Images**: Google-managed OS images (Debian, Ubuntu, CentOS, Windows Server, Container-Optimized OS)
- **Custom Images**: Your own image created from a running instance or disk
- **Snapshots**: Point-in-time backup of a persistent disk (incremental)
- **Machine Images**: Full backup of an instance including all disks and configuration

### Machine Types (Families)

| Family | Optimized For | Example |
|--------|--------------|---------|
| `e2` | Cost-optimized general-purpose | e2-medium (free tier eligible) |
| `n2`, `n2d` | Balanced general-purpose | n2-standard-4 |
| `c2`, `c3` | Compute-intensive | c2-standard-8 |
| `m1`, `m2`, `m3` | Memory-intensive | m1-ultramem-40 |
| `a2`, `a3` | GPU / ML training | a2-highgpu-1g |
| `t2d` | Scale-out, Arm-based | t2d-standard-4 |

### Zones and Regions
- **Zone**: A single data center location (e.g., `us-central1-a`)
- **Region**: A geographic area containing multiple zones (e.g., `us-central1`)
- Deploy across multiple zones for high availability

### SSH Keys
Access Linux VMs via SSH. GCP can manage SSH keys via OS Login (recommended) or project/instance metadata.

### Firewall Rules
VPC-level rules that control inbound/outbound traffic using tags or service accounts as targets.

---

## CLI Commands

### Create an Instance
```bash
# Basic VM
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud

# With startup script
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx'

# With a service account attached
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --machine-type=n2-standard-2 \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --service-account=my-app@my-project.iam.gserviceaccount.com \
  --scopes=cloud-platform
```

### Manage Instances
```bash
gcloud compute instances list                          # List all VMs
gcloud compute instances list --filter="zone:us-central1-a AND status=RUNNING"

gcloud compute instances start my-vm --zone=us-central1-a
gcloud compute instances stop my-vm --zone=us-central1-a
gcloud compute instances reset my-vm --zone=us-central1-a    # Hard reboot
gcloud compute instances delete my-vm --zone=us-central1-a

# Describe instance details
gcloud compute instances describe my-vm --zone=us-central1-a
```

### SSH Into an Instance
```bash
# gcloud manages SSH keys automatically via OS Login
gcloud compute ssh my-vm --zone=us-central1-a

# SSH with a tunnel (port forwarding)
gcloud compute ssh my-vm --zone=us-central1-a -- -L 5432:localhost:5432

# Run a remote command
gcloud compute ssh my-vm --zone=us-central1-a --command="sudo systemctl status nginx"
```

### Copy Files To/From Instance
```bash
# Upload to VM
gcloud compute scp ./local-file.txt my-vm:~/remote-file.txt --zone=us-central1-a

# Download from VM
gcloud compute scp my-vm:~/remote-file.txt ./local-file.txt --zone=us-central1-a

# Recursive copy
gcloud compute scp --recurse ./local-dir my-vm:~/remote-dir --zone=us-central1-a
```

---

## Disks

```bash
# List disks
gcloud compute disks list

# Create a persistent disk
gcloud compute disks create my-data-disk \
  --size=100GB \
  --type=pd-ssd \
  --zone=us-central1-a

# Attach disk to a running VM
gcloud compute instances attach-disk my-vm \
  --disk=my-data-disk \
  --zone=us-central1-a

# Detach disk
gcloud compute instances detach-disk my-vm \
  --disk=my-data-disk \
  --zone=us-central1-a

# Create a snapshot of a disk
gcloud compute snapshots create my-snapshot \
  --source-disk=my-disk \
  --source-disk-zone=us-central1-a
```

Disk types:

| Type | Description | Use Case |
|------|-------------|----------|
| `pd-standard` | HDD — low cost | Bulk storage, backups |
| `pd-balanced` | SSD — balanced | General workloads |
| `pd-ssd` | High-performance SSD | Databases, latency-sensitive |
| `pd-extreme` | Highest IOPS SSD | Large OLTP databases |
| `hyperdisk-extreme` | Newest, configurable IOPS | Enterprise databases |

---

## Firewall Rules

```bash
# Allow HTTP and HTTPS traffic to VMs tagged "web-server"
gcloud compute firewall-rules create allow-http \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --target-tags=web-server

# Allow SSH only from your IP
gcloud compute firewall-rules create allow-ssh-from-my-ip \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=YOUR_IP/32 \
  --target-tags=allow-ssh

# Apply a tag to your instance
gcloud compute instances add-tags my-vm \
  --tags=web-server,allow-ssh \
  --zone=us-central1-a

# List all firewall rules
gcloud compute firewall-rules list
```

---

## Instance Groups (Auto Scaling)

### Managed Instance Groups (MIG) — equivalent to Auto Scaling Groups
```bash
# 1. Create an instance template
gcloud compute instance-templates create my-template \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=web-server \
  --metadata=startup-script='#!/bin/bash
    apt-get install -y nginx && systemctl start nginx'

# 2. Create a Managed Instance Group
gcloud compute instance-groups managed create my-mig \
  --base-instance-name=my-vm \
  --template=my-template \
  --size=3 \
  --zone=us-central1-a

# 3. Enable autoscaling
gcloud compute instance-groups managed set-autoscaling my-mig \
  --zone=us-central1-a \
  --min-num-replicas=2 \
  --max-num-replicas=10 \
  --target-cpu-utilization=0.6

# List instances in a group
gcloud compute instance-groups managed list-instances my-mig --zone=us-central1-a

# Rolling update to a new template
gcloud compute instance-groups managed rolling-action start-update my-mig \
  --version=template=my-new-template \
  --zone=us-central1-a
```

---

## OS Login (Recommended SSH Management)

OS Login ties SSH access to IAM roles — no need to manage SSH keys manually in metadata.

```bash
# Enable OS Login at project level
gcloud compute project-info add-metadata --metadata=enable-oslogin=TRUE

# Grant a user SSH access
gcloud projects add-iam-policy-binding my-project \
  --member="user:alice@example.com" \
  --role="roles/compute.osLogin"

# Grant sudo access
gcloud projects add-iam-policy-binding my-project \
  --member="user:alice@example.com" \
  --role="roles/compute.osAdminLogin"
```

---

## Startup Scripts vs Cloud-Init

```bash
# Startup script via metadata (runs on every boot)
gcloud compute instances create my-vm \
  --metadata=startup-script-url=gs://my-bucket/startup.sh

# Using cloud-init (runs once on first boot — more powerful)
gcloud compute instances create my-vm \
  --metadata=user-data='#cloud-config
  packages:
    - nginx
    - git
  runcmd:
    - systemctl enable nginx
    - systemctl start nginx'
```

---

## Pricing Tips

- **Sustained use discounts**: Automatically applied when you run a VM for more than 25% of a month — up to 30% discount, no commitment needed.
- **Committed use discounts (CUDs)**: 1- or 3-year commitments — up to 57% discount.
- **Spot VMs**: Up to 91% cheaper — but can be preempted with 30s notice. Great for batch jobs and fault-tolerant workloads.
- **E2 machines**: Cheapest general-purpose option for dev/test workloads.

```bash
# Create a Spot VM
gcloud compute instances create my-spot-vm \
  --zone=us-central1-a \
  --machine-type=n2-standard-4 \
  --provisioning-model=SPOT \
  --instance-termination-action=DELETE
```