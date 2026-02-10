# Disk and Memory Management

Comprehensive guide to managing disk space, partitions, filesystems, memory, and swap in Linux.

## Table of Contents
- [Filesystem Usage](#filesystem-usage-df)
- [Disk Usage](#disk-usage-du)
- [Memory (RAM)](#memory-ram)
- [Block Devices and Partitions](#block-devices-and-partitions)
- [Creating New Partitions](#creating-new-partitions-workflow)
- [Filesystem Operations](#filesystem-operations)
- [Swap Partition](#swap-partition)
- [Best Practices](#best-practices)

---

## Filesystem Usage (df)

Display filesystem disk space usage, available space, and mount points.

```bash
df                      # Show filesystem info (blocks)
df -h                   # Human-readable format (GB, MB)
df -T                   # Include filesystem type
df -a                   # All filesystems including pseudo/virtual ones
df -h /path             # Disk usage of specific mount point
df -i                   # Show inode usage ⚠️ Important for troubleshooting
```

### Understanding df Output

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   45G   55G  45% /
/dev/sdb1       500G  200G  300G  40% /data
```

### Inode Exhaustion Problem

**Scenario:** Disk shows free space, but you can't create files.

```bash
df -h           # Shows plenty of free space
df -i           # Shows 100% inode usage ⚠️
```

**Cause:** Too many small files have exhausted available inodes (file metadata structures).

**Solution:** Delete unnecessary files or restructure data storage.

---

## Disk Usage (du)

Calculate disk space used by files and directories.

```bash
du -h                   # Human-readable format
du -sh <path>           # Total size of directory (summary)
du -a                   # Show all files + directories
du -a <path>            # All accessible items with sizes
du --time               # Include last modification time
du -xh                  # Stay on same filesystem only ⚠️ Important for servers
```

### Examples

```bash
# Find total size of /home
du -sh /home

# Find largest directories in current location
du -h --max-depth=1 | sort -hr

# Find disk usage with timestamps
du -sh --time /var/log/*
```

### Staying on Same Filesystem

```bash
du -xh /              # Don't cross filesystem boundaries
                      # Important when dealing with mounted drives
```

---

## Memory (RAM)

Display RAM usage including used, free, shared, and cached memory.

```bash
free -h               # Human-readable memory usage
free -m               # Show in megabytes
free -g               # Show in gigabytes
free -s 2             # Update every 2 seconds
```

### Example Output

```
              total        used        free      shared  buff/cache   available
Mem:           15Gi       8.2Gi       1.3Gi       524Mi       6.1Gi       6.8Gi
Swap:         2.0Gi       128Mi       1.9Gi
```

**Key fields:**
- `total` - Total installed RAM
- `used` - Memory in use by processes
- `free` - Unused memory
- `buff/cache` - Memory used for caching (can be freed if needed)
- `available` - Memory available for starting new applications

---

## Block Devices and Partitions

### lsblk - List Block Devices

Display disks, partitions, and mount points in tree format.

```bash
lsblk                   # List all block devices
lsblk -f                # Include filesystem type + UUID + mount point
```

#### Example Output

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  447G  0 disk
├─sda1   8:1    0  100M  0 part /boot
├─sda2   8:2    0   16G  0 part [SWAP]
├─sda3   8:3    0  430G  0 part /
└─sda4   8:4    0  500M  0 part
```

**Legend:**
- `sda` → Physical disk
- `sda1`, `sda2`, `sda3`, `sda4` → Partitions

### fdisk - Partition Table Manipulator

Traditional partitioning tool for disks up to 2TB.

```bash
sudo fdisk -l           # List all disks and partitions (size, type, sectors)
sudo fdisk /dev/sdb     # Open disk for partitioning (interactive mode)
```

#### Common fdisk Commands (Interactive Mode)

| Command | Action |
|---------|--------|
| `p` | Print partition table |
| `n` | Create new partition |
| `d` | Delete partition |
| `t` | Change partition type |
| `w` | Write changes and exit |
| `q` | Quit without saving |

⚠️ **Warning:** Changes apply only after pressing `w` (write).

### parted - Modern Partitioning Tool

Better support for large disks (>2TB) and GPT partition tables.

```bash
sudo parted             # Open interactive shell
sudo parted /dev/sdb    # Open specific disk
```

#### Common parted Commands

```bash
print                   # Show partitions
mkpart                  # Create partition
rm                      # Delete partition
quit                    # Exit
```

### Mount Information

```bash
mount                   # Show currently mounted filesystems
findmnt                 # Cleaner mount hierarchy view
```

---

## Creating New Partitions (Workflow)

Complete step-by-step process to create and use a new partition.

### Step 1: Check Available Disks

```bash
lsblk                   # Identify disk and free space (Always first!)
```

### Step 2: Open Partitioning Tool

```bash
sudo fdisk /dev/sdb     # For disks < 2TB
# OR
sudo parted /dev/sdb    # For disks > 2TB (recommended)
```

### Step 3: Create Partition (Using fdisk)

Inside fdisk interactive mode:

```
Command (m for help): p
# Shows existing partitions and confirms free space

Command (m for help): n
# Create new partition

Partition type: p (primary) or l (logical)
Partition number: <press Enter for default>
First sector: <press Enter for default>
Last sector or +size: +10G
# Example: +10G creates a 10GB partition
# fdisk auto-aligns sectors
```

### Step 4: Write Changes

```
Command (m for help): w
# Writes partition table and exits
```

⚠️ **Critical:** Changes apply ONLY after pressing `w`.

### Step 5: Create Filesystem

```bash
sudo mkfs.ext4 /dev/sdb5        # Create ext4 filesystem
# OR
sudo mkfs.xfs /dev/sdb5         # Create XFS filesystem
```

### Step 6: Create Mount Directory

```bash
sudo mkdir /mnt/newdisk         # Create mount point
```

### Step 7: Mount the Partition

```bash
sudo mount /dev/sdb5 /mnt/newdisk   # Mount partition
```

### Step 8: Verify

```bash
df -h && lsblk                  # Verify mount point and size
```

---

## Filesystem Operations

### Making Mount Permanent (fstab)

To automatically mount partition on boot, add entry to `/etc/fstab`.

#### Step 1: Get UUID

```bash
blkid                           # Copy UUID of partition
```

#### Step 2: Edit fstab

```bash
sudo vi /etc/fstab
```

Add entry:
```
UUID=xxxx-xxxx-xxxx /data ext4 defaults 0 2
```

**Field explanation:**
1. `UUID=xxxx` - Partition identifier
2. `/data` - Mount point
3. `ext4` - Filesystem type
4. `defaults` - Mount options
5. `0` - Dump (backup) - 0=no, 1=yes
6. `2` - fsck order (0=skip, 1=root, 2=other)

#### Step 3: Test fstab Entry

```bash
sudo mount -a               # Mount all filesystems in fstab
# If no error → fstab is correct
```

### Unmounting and Cleanup

```bash
# Unmount filesystem (must unmount before deletion)
sudo umount /mnt/newdisk

# Delete partition using fdisk
sudo fdisk /dev/sdb
# In fdisk:
d                           # Delete
<partition number>          # Select partition
w                           # Write changes
```

### Filesystem Check (fsck)

Check and repair filesystem integrity.

```bash
sudo fsck /dev/sdb5         # Check filesystem
sudo fsck -y /dev/sdb5      # Auto-fix detected issues
```

⚠️ **Critical Rules:**
- Run `fsck` ONLY on unmounted filesystems
- Always back up data before making partition changes
- This is CRITICAL in production environments

---

## Swap Partition

Swap acts as logical backup of RAM, used when physical memory is exhausted. Helps prevent OOM (Out Of Memory) kills.

### Creating Swap Partition

#### Step 1: Identify Disk

```bash
lsblk -f                    # Check available disks and free space
```

#### Step 2: Create Partition Using fdisk

```bash
sudo fdisk /dev/sda
```

Inside fdisk:

```
n                           # New partition
p                           # Primary (recommended)
<partition number>          # e.g., 5
<First sector>              # Press Enter for default
<Last sector>               # Example: +2G for 2GB swap

t                           # Change partition type
<partition number>          # Select partition
82                          # 82 = Linux swap

w                           # Write changes
```

#### Step 3: Format as Swap

```bash
sudo mkswap /dev/sda5       # Mark partition as swap space
```

#### Step 4: Enable Swap

```bash
sudo swapon /dev/sda5       # Activate swap
```

#### Step 5: Verify Swap

```bash
swapon --show               # Show active swap
# OR
free -h                     # Check memory + swap
```

### Making Swap Permanent

Edit `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Add entry:
```
/dev/sda5 none swap sw 0 0
```

**Field explanation:**
- `/dev/sda5` - Swap partition
- `none` - No mount point (swap doesn't mount)
- `swap` - Filesystem type
- `sw` - Enable swap
- `0 0` - No dump/fsck

#### Test fstab Entry

```bash
sudo swapoff -a             # Disable all swap
sudo swapon -a              # Enable swap from fstab
# If no error → fstab is correct
```

### Disabling Swap

```bash
sudo swapoff /dev/sda5      # Disable specific swap partition
sudo swapoff -a             # Disable all swap
```

**Use cases:** Resizing swap, deleting swap partition

---

## Swap File (Alternative to Partition)

Swap files are easier to resize than partitions and preferred in cloud environments.

### Creating Swap File

```bash
# Create 2GB file
sudo fallocate -l 2G /swapfile

# Set correct permissions (security)
sudo chmod 600 /swapfile

# Format as swap
sudo mkswap /swapfile

# Enable swap
sudo swapon /swapfile

# Verify
free -h
```

### Making Swap File Permanent

Add to `/etc/fstab`:
```
/swapfile none swap sw 0 0
```

### Advantages of Swap Files
- ✅ Easier to create
- ✅ Easier to resize
- ✅ Preferred in cloud/virtual environments
- ✅ No partitioning required

---

## Monitoring Swap Usage

```bash
vmstat                      # Virtual memory statistics
vmstat 1                    # Update every 1 second

htop                        # Interactive process viewer with swap info

free -h                     # Simple memory + swap view
```

---

## Quick Reference: Partitioning Workflow

1. **Check disks:** `lsblk`
2. **Open partition tool:** `fdisk` or `parted`
3. **Create partition:** `n` (new)
4. **Write changes:** `w`
5. **Create filesystem:** `mkfs.ext4`
6. **Create mount point:** `mkdir`
7. **Mount:** `mount`
8. **Verify:** `df -h && lsblk`
9. **Make permanent:** Edit `/etc/fstab`

---

## Best Practices

### Disk and Partition Management

1. **Always check free space** with `lsblk` before partitioning
2. **Backup data** before making partition changes
3. **Test fstab entries** with `mount -a` before rebooting
4. **Use UUID** in fstab instead of device names (e.g., `/dev/sda1`)
   - Device names can change; UUIDs don't
5. **Unmount before fsck** - never run filesystem check on mounted partition
6. **Use appropriate filesystem:**
   - `ext4` - General purpose, stable
   - `xfs` - Large files, high performance
   - `btrfs` - Advanced features, snapshots

### Memory and Swap

1. **Monitor swap usage** regularly - high swap use indicates RAM shortage
2. **Swap size recommendations:**
   - RAM < 2GB: Swap = 2x RAM
   - RAM 2-8GB: Swap = 1x RAM
   - RAM > 8GB: Swap = 8GB (or based on hibernation needs)
3. **Use swap files in cloud** environments for flexibility
4. **Set correct permissions** on swap files (`chmod 600`)

### Monitoring and Troubleshooting

1. **Check inode usage** (`df -i`) when disk is full but space shows available
2. **Use `du -xh`** to stay on same filesystem when analyzing disk usage
3. **Monitor I/O** with `iostat` for performance issues
4. **Regular filesystem checks** prevent data corruption
5. **Keep 10-20% free space** on partitions for optimal performance

---

## Common Issues and Solutions

### Issue: Cannot Create Files Despite Free Space

**Diagnosis:**
```bash
df -h           # Shows free space
df -i           # Shows 100% inode usage ⚠️
```

**Solution:** Delete unnecessary files or restructure storage.

### Issue: Partition Not Mounting on Boot

**Diagnosis:**
```bash
cat /etc/fstab  # Check fstab entry
sudo mount -a   # Test mount
journalctl -b   # Check boot logs
```

**Solution:** Fix fstab syntax, verify UUID, check filesystem type.

### Issue: High Swap Usage

**Diagnosis:**
```bash
free -h         # Check swap usage
vmstat 1        # Monitor swap in/out
```

**Solution:**
- Add more RAM
- Optimize applications
- Investigate memory leaks

---

## Filesystem Types Comparison

| Filesystem | Use Case | Advantages | Limitations |
|------------|----------|------------|-------------|
| `ext4` | General purpose | Stable, mature, journaling | No snapshots |
| `xfs` | Large files, databases | High performance, scalable | Hard to shrink |
| `btrfs` | Advanced features | Snapshots, compression, RAID | Less mature |
| `ntfs` | Windows compatibility | Cross-platform | Limited Linux support |
| `vfat/FAT32` | USB drives | Universal compatibility | 4GB file size limit |

---

This guide covers essential disk, memory, and filesystem management for Linux system administration and DevOps operations.