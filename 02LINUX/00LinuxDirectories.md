# Linux Directory Structure (Filesystem Hierarchy Standard)

Comprehensive guide to the Linux filesystem hierarchy, explaining every major directory and its purpose.

## Table of Contents
- [Overview](#overview)
- [Root Directory (/)](#root-directory-)
- [Essential Binaries](#essential-binaries)
- [Boot Files](#boot-files)
- [Device Files](#device-files)
- [System Configuration](#system-configuration)
- [User Directories](#user-directories)
- [Libraries and Modules](#libraries-and-modules)
- [Mount Points](#mount-points)
- [Optional Software](#optional-software)
- [Virtual Filesystems](#virtual-filesystems)
- [Runtime Data](#runtime-data)
- [Temporary Files](#temporary-files)
- [User System Resources](#user-system-resources)
- [Variable Data](#variable-data)
- [Service Data](#service-data)

---

## Overview

### Complete Directory Tree

```
/
├── bin         → Essential user binaries
├── boot        → Boot loader files
├── dev         → Device files
├── etc         → System configuration
├── home        → User home directories
├── lib         → Shared libraries
├── lib64       → 64-bit libraries
├── media       → Removable media mount points
├── mnt         → Temporary mount points
├── opt         → Optional software
├── proc        → Process information (virtual)
├── root        → Root user home directory
├── run         → Runtime data
├── sbin        → System binaries
├── srv         → Service data
├── sys         → System information (virtual)
├── tmp         → Temporary files
├── usr         → User programs and data
│   ├── bin     → User binaries
│   ├── sbin    → System admin binaries
│   ├── lib     → Libraries
│   └── local   → Locally installed software
└── var         → Variable data (logs, caches)
```

---

## Root Directory (/)

**The root of the filesystem — everything starts here.**

All directories and files in Linux originate from `/`. This is fundamentally different from Windows (C:\, D:\).

### Key Characteristics

- **Single unified tree** - All storage appears under one root
- **Case-sensitive** - `/Home` and `/home` are different
- **No drive letters** - Disks are mounted as subdirectories
- **Everything is a file** - Devices, processes, configs are all files

---

## Essential Binaries

### /bin - Essential User Binaries

**Essential command-line programs required for basic system operation.**

Modern distributions often symlink `/bin` → `/usr/bin`.

#### Categories and Commands

**File & directory operations:**
```bash
ls, cp, mv, rm, mkdir, rmdir, pwd, touch, ln
```

**File viewing & manipulation:**
```bash
cat, tac, head, tail, sort, uniq, wc, less, more
```

**Text processing:**
```bash
grep, sed, awk, cut, tr
```

**Shell & scripting:**
```bash
bash          # GNU Bourne Again Shell
sh            # POSIX-compliant shell
dash          # Lightweight shell (Ubuntu default)
```

**User/session commands:**
```bash
login, logout, su         # Switch user
```

**System & environment utilities:**
```bash
echo, date, hostname, uname
true          # Return success (exit 0)
false         # Return failure (exit 1)
```

**Compression & archiving:**
```bash
tar, gzip, gunzip, zcat
```

#### /bin vs /usr/bin

**Historical difference:**
- `/bin` - Essential for booting and single-user mode
- `/usr/bin` - Additional user commands

**Modern systems:**
```bash
ls -ld /bin
# lrwxrwxrwx 1 root root 7 Jan 1 00:00 /bin -> usr/bin
```

Most modern distributions symlink `/bin` → `/usr/bin` for simplicity.

---

### /sbin - System Binaries

**System administration binaries used by root/sudoers for system management.**

#### User & Group Administration

```bash
useradd, usermod, userdel
groupadd, groupmod, groupdel
```

#### Filesystem & Disk Management

```bash
/sbin/fsck              # Filesystem consistency check
/sbin/fsck.ext4         # Check ext4 filesystem
/sbin/mkfs              # Create filesystem
/sbin/mkfs.ext4         # Format ext4 filesystem
/sbin/mount             # Mount filesystem
/sbin/umount            # Unmount filesystem
/sbin/blkid             # Show block device attributes
```

#### Boot & System Control

```bash
/sbin/reboot            # Reboot system
/sbin/poweroff          # Shut down system
/sbin/halt              # Stop system
/sbin/init              # System initialization (legacy)
/sbin/shutdown          # Schedule shutdown/reboot
```

#### Networking & Firewall

```bash
/sbin/iptables          # Configure kernel firewall
/sbin/ip                # Network configuration tool
/sbin/route             # Show/modify routing table (legacy)
/sbin/ifconfig          # Configure interfaces (legacy)
```

#### Hardware and Kernel Utilities

```bash
/sbin/lsblk             # List block devices
/sbin/fdisk             # Disk partitioning
/sbin/parted            # Advanced partition tool
/sbin/swapon            # Enable swap
/sbin/swapoff           # Disable swap
/sbin/sysctl            # Modify kernel parameters
```

#### System Services and Modules

```bash
/sbin/modprobe          # Load/unload kernel modules
/sbin/lsmod             # List loaded kernel modules
/sbin/service           # Manage services (SysVinit compatibility)
```

#### /sbin vs /usr/sbin

- `/sbin` → Essential admin binaries
- `/usr/sbin` → Non-essential admin binaries
- Modern systems often symlink `/sbin` → `/usr/sbin`

---

## Boot Files

### /boot - Boot Loader and Kernel Files

**Files required to boot the Linux system.**

⚠️ **Critical:** If `/boot` is broken, the system may not boot.

#### Directory Structure

```
/boot
├── vmlinuz-6.5.0-14-generic        # Linux kernel image
├── vmlinuz-6.2.0-39-generic        # Older kernel (fallback)
├── initrd.img-6.5.0-14-generic     # Initial RAM filesystem
├── initrd.img-6.2.0-39-generic     # Older initrd
├── System.map-6.5.0-14-generic     # Kernel symbol table
├── config-6.5.0-14-generic         # Kernel build configuration
├── abi-6.5.0-14-generic            # Kernel ABI (Ubuntu-specific)
├── memtest86+.bin                  # Memory testing utility
├── grub/                           # GRUB bootloader
│   ├── grub.cfg                    # Auto-generated boot menu
│   ├── grubenv                     # GRUB environment variables
│   ├── fonts/
│   ├── locale/
│   └── x86_64-efi/                 # GRUB modules
└── efi/                            # EFI bootloader files
    └── EFI/
        ├── ubuntu/
        │   ├── grubx64.efi         # GRUB EFI binary
        │   ├── shimx64.efi         # Secure Boot shim
        │   └── mmx64.efi
        └── BOOT/
            └── BOOTX64.EFI         # Fallback bootloader
```

#### Key Files Explained

**Kernel files:**
- `vmlinuz-*` - Compressed Linux kernel image
- `initrd.img-*` - Initial RAM filesystem (loads before root FS)
- `System.map-*` - Kernel symbol table for debugging
- `config-*` - Kernel build configuration options

**GRUB files:**
- `grub.cfg` - Auto-generated GRUB menu (don't edit directly!)
- `grubenv` - GRUB environment variables
- `*.mod` - GRUB modules for features

**EFI files:**
- `grubx64.efi` - GRUB for UEFI systems
- `shimx64.efi` - Secure Boot shim
- `BOOTX64.EFI` - Fallback bootloader

---

## Device Files

### /dev - Device Files

**Device files representing hardware and virtual devices.**

⚠️ **Important:** These are NOT real files — they're interfaces to kernel drivers. No data stored on disk.

Managed by `udev` - devices appear/disappear automatically.

#### Block Devices (Storage)

```bash
/dev/sda            # First SATA/SCSI disk
/dev/sda1           # First partition on sda
/dev/sda2           # Second partition on sda
/dev/sdb            # Second SATA/SCSI disk
/dev/nvme0n1        # NVMe SSD device
/dev/nvme0n1p1      # First partition on NVMe
/dev/loop0          # Loopback device (ISO, snaps)
```

#### Virtual Memory Devices

```bash
/dev/ram0           # RAM disk
```

#### Terminal Devices

```bash
/dev/tty            # Current terminal
/dev/tty0           # System console
/dev/tty1           # Virtual terminal 1
/dev/tty2           # Virtual terminal 2
/dev/ttyS0          # Serial port
```

#### Pseudo Terminals

Used by SSH and terminal emulators.

```bash
/dev/pts/0          # SSH terminal session
/dev/pts/1          # Another pseudo-terminal
/dev/pts/2          # Terminal emulator tab
```

#### Special Virtual Devices

```bash
/dev/null           # Discards all data (black hole)
/dev/zero           # Outputs infinite zero bytes
/dev/random         # Cryptographic random data (blocking)
/dev/urandom        # Non-blocking random data
/dev/full           # Always returns "disk full" error
```

**Common uses:**
```bash
# Discard output
command > /dev/null 2>&1

# Generate random password
head -c 32 /dev/urandom | base64

# Create empty file
dd if=/dev/zero of=file.img bs=1M count=100
```

#### System Control Devices

```bash
/dev/console        # System console
/dev/kmsg           # Kernel log messages
```

#### Shared Memory

Used by browsers, databases, containers.

```bash
/dev/shm            # Shared memory (tmpfs)
```

#### Stable Disk Identifiers

**Recommended for `/etc/fstab`:**

```bash
/dev/disk/by-uuid/          # Stable UUIDs
/dev/disk/by-label/         # Disk labels
/dev/disk/by-id/            # Hardware IDs
/dev/disk/by-path/          # Physical paths
```

**Example:**
```bash
ls -l /dev/disk/by-uuid/
# lrwxrwxrwx 1 root root 10 Jan 15 10:00 abc123... -> ../../sda1
```

#### Device Mapper

Used by LVM, dm-crypt, Docker, Kubernetes.

```bash
/dev/mapper/vg0-root        # Logical volume
/dev/mapper/luks-xyz        # Encrypted volume
```

---

## System Configuration

### /etc - System Configuration Files

**System-wide configuration files. No binaries, no user data — only plain text configurations.**

⚠️ **Golden Rules:**
- ✅ Back up before editing
- ✅ Use comments to document changes
- ✅ Restart/reload services after changes
- ❌ Avoid deleting files

#### Directory Structure

```
/etc
├── passwd              # User accounts
├── shadow              # Encrypted passwords
├── group               # Group definitions
├── gshadow             # Secure group passwords
├── sudoers             # Sudo permissions
├── sudoers.d/          # Modular sudo rules
│
├── hostname            # System hostname
├── hosts               # Local DNS overrides
├── resolv.conf         # DNS resolver config
│
├── fstab               # Filesystems mounted at boot
├── crypttab            # Encrypted volumes
│
├── os-release          # OS information
├── lsb-release         # Distribution info
│
├── profile             # Global environment
├── profile.d/          # Per-app env scripts
├── bash.bashrc         # Global bash config
│
├── ssh/                # SSH configuration
│   ├── sshd_config     # SSH server config
│   ├── ssh_config      # SSH client config
│   └── ssh_host_*_key  # Host keys
│
├── systemd/            # Systemd configuration
│   ├── system/         # System units
│   └── user/           # User units
│
├── init.d/             # Init scripts (legacy)
├── cron.d/             # Cron jobs
├── cron.daily/         # Daily cron scripts
├── cron.hourly/        # Hourly cron scripts
│
├── logrotate.conf      # Log rotation config
├── logrotate.d/        # Per-service rotation
│
├── apt/                # APT config (Debian/Ubuntu)
│   ├── sources.list
│   └── sources.list.d/
│
├── yum.repos.d/        # YUM repos (RHEL/CentOS)
│
├── network/            # Network config
├── netplan/            # Netplan (Ubuntu)
│
├── sysctl.conf         # Kernel parameters
├── sysctl.d/           # Modular sysctl
│
├── services            # Port ↔ service mapping
└── issue               # Login banner
```

#### Core Identity and Security Files

**User accounts:**
```bash
/etc/passwd     # username:x:UID:GID:fullname:/home:shell
# Example: john:x:1001:1001:John Doe:/home/john:/bin/bash

/etc/shadow     # Encrypted passwords + aging (root only)
/etc/group      # Group definitions
/etc/gshadow    # Secure group passwords
```

**Sudo configuration:**
```bash
/etc/sudoers            # Main sudo config
/etc/sudoers.d/         # Modular sudo rules

# Never edit directly - use:
sudo visudo
```

#### Networking and DNS

```bash
/etc/hostname           # System hostname
/etc/hosts              # Local DNS overrides
# Example:
# 127.0.0.1 localhost
# 192.168.1.10 server1

/etc/resolv.conf        # DNS resolver
# nameserver 8.8.8.8
# nameserver 1.1.1.1

/etc/services           # Port ↔ service mapping
```

#### Boot, Disks, and Mounts

```bash
/etc/fstab              # Filesystems mounted at boot
# UUID=xxx / ext4 defaults 0 1

/etc/crypttab           # Encrypted volumes
```

#### Shell and Environment

Applied to all users:

```bash
/etc/profile            # Global environment variables
/etc/profile.d/         # Per-application env scripts
/etc/bash.bashrc        # Global bash configuration
```

#### SSH Configuration

```bash
/etc/ssh/sshd_config    # SSH server config
# Port 22
# PermitRootLogin no
# PasswordAuthentication yes

/etc/ssh/ssh_config     # SSH client config
/etc/ssh/ssh_host_*     # Host keys
```

#### Systemd and Services

```bash
/etc/systemd/system/    # Custom service units
/etc/systemd/user/      # User-level services

# Overrides go here instead of /lib/systemd
```

#### Scheduled Jobs

```bash
/etc/crontab            # System crontab
/etc/cron.d/            # Per-application cron jobs
/etc/cron.daily/        # Daily scripts
/etc/cron.hourly/       # Hourly scripts
/etc/cron.weekly/       # Weekly scripts
/etc/cron.monthly/      # Monthly scripts
```

#### Logging and Rotation

Prevent log files from filling the disk:

```bash
/etc/logrotate.conf     # Main config
/etc/logrotate.d/       # Per-service config
```

#### Package Manager Configurations

**Debian/Ubuntu:**
```bash
/etc/apt/sources.list           # APT repositories
/etc/apt/sources.list.d/        # Additional repos
```

**RHEL/CentOS:**
```bash
/etc/yum.repos.d/               # YUM repositories
```

#### Kernel and System Tuning

```bash
/etc/sysctl.conf                # Kernel parameters
/etc/sysctl.d/                  # Modular sysctl configs

# Example tuning:
# net.ipv4.ip_forward=1
# vm.swappiness=10
```

#### Informational Files

```bash
/etc/os-release         # OS information
/etc/lsb-release        # Distribution info
/etc/issue              # Pre-login banner
/etc/motd               # Message of the day
```

---

## User Directories

### /home - User Home Directories

**Home directories for normal users.**

Format: `/home/<username>`

#### Typical User Home Structure

```
/home/alice
├── .bashrc                     # Interactive shell config
├── .bash_profile               # Login shell config
├── .profile                    # POSIX login fallback
├── .bash_history               # Command history
├── .bash_logout                # Logout commands
│
├── .ssh/                       # SSH configuration (700)
│   ├── authorized_keys         # Allowed SSH keys (600)
│   ├── id_ed25519              # Private key (600)
│   ├── id_ed25519.pub          # Public key
│   └── config                  # SSH client config
│
├── .config/                    # Application configs (XDG)
│   ├── git/
│   ├── vscode/
│   └── systemd/                # User services
│
├── .cache/                     # Application cache
├── .local/
│   ├── bin/                    # User executables (in PATH)
│   │   └── custom-script
│   └── share/                  # User app data
│
├── Documents/
├── Downloads/
├── Pictures/
├── Videos/
├── Music/
└── scripts/                    # User scripts
```

#### Important Subdirectories

**Shell & environment config:**
```bash
.bashrc             # User shell aliases, functions, env vars
.bash_profile       # Login shell settings
.profile            # POSIX shell login config
.bash_logout        # Commands run on logout
```

**SSH configuration:**
```bash
.ssh/
├── authorized_keys     # Public keys allowed to login
├── id_rsa              # Private SSH key (keep secure!)
├── id_rsa.pub          # Public SSH key
└── config              # Per-user SSH settings

# Correct permissions:
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 600 ~/.ssh/authorized_keys
```

**History and session files:**
```bash
.bash_history           # Command history
.lesshst                # less command history
.viminfo                # vim session data
```

**Application config (XDG standard):**
```bash
.config/                # Modern app configs
.cache/                 # App cache files
.local/
├── bin/                # User-installed executables
└── share/              # User app data
```

**Note:** `/home/<user>/.local/bin` is often added to `$PATH`.

**User data directories:**

Created automatically by desktop environments:
```bash
Desktop/
Documents/
Downloads/
Music/
Pictures/
Videos/
```

---

### /root - Root User Home

**Home directory for the root (superuser) account.**

⚠️ **Important:** `/root` is NOT the same as `/`

#### Structure

```
/root
├── .bashrc                     # Root shell config
├── .bash_profile
├── .bash_history               # Root command history
│
├── .ssh/                       # Root SSH config
│   ├── authorized_keys
│   ├── id_rsa
│   └── config
│
├── .config/                    # Root app configs
│
├── scripts/                    # Administrative scripts
│   ├── backup.sh
│   ├── firewall.sh
│   ├── cleanup.sh
│   └── monitor.sh
│
├── logs/                       # Admin-only logs
│   └── admin.log
│
├── tmp/                        # Root-only temp files
│
└── tools/                      # Recovery/maintenance tools
    ├── disk-check.sh
    └── restore.sh
```

#### Common Files

```bash
.bashrc                 # Root shell configuration
.bash_profile           # Root login settings
.bash_history           # Root command history
.ssh/                   # Root SSH keys and config
scripts/                # Administrative automation
logs/                   # Custom admin logs
```

---

## Libraries and Modules

### /lib - Shared Libraries

**Shared libraries and kernel modules required for booting and running system commands.**

⚠️ **Critical:** If `/lib` is broken, the system may not boot.

#### Directory Structure

```
/lib
├── modules/                    # Kernel modules
│   └── <kernel-version>/
│       ├── kernel/             # Module files (.ko)
│       │   ├── drivers/
│       │   │   ├── net/        # Network drivers
│       │   │   ├── block/      # Disk drivers
│       │   │   ├── usb/        # USB drivers
│       │   │   ├── gpu/        # Graphics drivers
│       │   │   ├── input/      # Input devices
│       │   │   └── sound/      # Audio drivers
│       │   └── fs/             # Filesystems
│       │       ├── ext4/
│       │       ├── xfs/
│       │       ├── nfs/
│       │       └── overlayfs/
│       ├── modules.dep         # Dependency map
│       ├── modules.alias       # Device → module mapping
│       └── modules.builtin     # Built-in modules
│
├── firmware/                   # Hardware firmware
│   ├── intel/
│   ├── amd/
│   └── iwlwifi/
│
├── systemd/                    # Core systemd
├── udev/                       # Device management
├── security/                   # PAM modules
│
├── ld-linux-x86-64.so.2        # Dynamic linker
├── libc.so.6                   # GNU C Library
├── libpthread.so.0             # Threading
├── libdl.so.2                  # Dynamic linking
└── libm.so.6                   # Math library
```

#### Key Components

**Kernel modules:**
- Drivers for hardware
- Filesystem support
- Network protocols

**Essential libraries:**

Almost every program depends on these:
```bash
libc.so.6               # GNU C Library (glibc)
ld-linux-x86-64.so.2    # Dynamic linker/loader
libpthread.so.0         # POSIX threads
libdl.so.2              # Dynamic linking
libm.so.6               # Math functions
```

**Firmware:**
```bash
/lib/firmware/          # Hardware firmware loaded by kernel
```

---

### /lib64 - 64-bit Libraries

**64-bit versions of essential libraries.**

Often symlinked or merged with `/lib` on modern systems.

---

## Mount Points

### /media - Removable Media

**Auto-mounted removable devices (USB, CD-ROM, SD cards).**

✅ Managed by `udev` + `systemd`
✅ Auto-created directories
❌ Not for permanent mounts

#### Structure

```
/media
├── alice/
│   ├── USB_DRIVE
│   ├── SanDisk_64GB
│   └── MyBackupDisk
│
├── bob/
│   └── KingstonUSB
│
└── root/
    └── ExternalHDD
```

#### What Happens When You Plug a USB?

1. Device detected (udev)
2. Filesystem identified
3. Mount point created: `/media/<user>/<LABEL>`
4. Device mounted automatically

**Example:** `/media/alice/USB_DRIVE`

#### Useful Commands

```bash
lsblk                           # See disks & mount points
mount | grep media
df -h | grep media

# Unmount safely
umount /media/alice/USB_DRIVE
```

---

### /mnt - Temporary Mounts

**Manual, temporary mount point for filesystems.**

Used for:
- Disks
- Partitions
- Network shares
- Cloud volumes

✅ Temporary
✅ Manual
❌ Not auto-managed

#### Structure

```
/mnt
├── data
├── backup
├── disk1
├── disk2
├── nfs
└── tmp_mount
```

Admins create these as needed.

#### Comparison

| Directory | Purpose |
|-----------|---------|
| `/media` | Auto-mounted removable devices |
| `/mnt` | Manual temporary mounts |
| `/etc/fstab` | Persistent mounts |

---

## Optional Software

### /opt - Optional Third-Party Software

**Self-contained, non-OS software. Vendor apps and custom enterprise tools.**

#### Characteristics

- Vendor applications
- Custom enterprise tools
- Manually installed services
- Self-contained (includes bin, lib, etc.)

#### Structure

```
/opt
├── google/
│   └── chrome/
│
├── oracle/
│   └── java/
│
├── splunk/
│   ├── bin
│   ├── etc
│   ├── var
│   └── lib
│
├── nginx/
│   ├── sbin
│   ├── conf
│   ├── logs
│   └── html
│
├── jenkins/
│   ├── bin
│   ├── war
│   ├── logs
│   └── plugins
│
└── custom-app/
    ├── bin
    ├── conf
    ├── lib
    ├── logs
    └── data
```

**Purpose:** Keeps third-party applications isolated from the OS.

---

## Virtual Filesystems

### /proc - Process Information

**Virtual filesystem exposing kernel and process information.**

⚠️ **Not real files:**
- Lives in RAM
- Generated dynamically by kernel
- Disappears on reboot
- Reading files = querying kernel state

#### Structure

```
/proc
├── cpuinfo             # CPU architecture & cores
├── meminfo             # RAM usage details
├── uptime              # System uptime
├── loadavg             # Load average
├── version             # Kernel version
├── cmdline             # Kernel boot parameters
├── filesystems         # Supported filesystems
├── mounts              # Mounted filesystems
│
├── sys/                # Kernel tuning
├── net/                # Network info
│
├── 1/                  # Process 1 (systemd/init)
├── 1234/               # Process 1234
├── 5678/               # Process 5678
│
└── self/               # Symlink to current process
```

#### Global System Info

```bash
cat /proc/cpuinfo               # CPU details
cat /proc/meminfo               # Memory usage
cat /proc/uptime                # Uptime in seconds
cat /proc/loadavg               # Load average
cat /proc/version               # Kernel version
cat /proc/cmdline               # Boot parameters
cat /proc/mounts                # Mounted filesystems
```

#### Per-Process Information

Each running process gets a directory: `/proc/<pid>`

```
/proc/1234/
├── cmdline             # Command used to start
├── environ             # Environment variables
├── cwd                 # Current working directory (symlink)
├── exe                 # Executable path (symlink)
├── fd/                 # Open file descriptors
├── maps                # Memory maps
├── status              # Process state, UID, memory
└── stat                # CPU usage, process stats
```

**Examples:**
```bash
# Current process
ls -l /proc/self

# Process command line
cat /proc/1234/cmdline

# Process environment
cat /proc/1234/environ

# Open files
ls -l /proc/1234/fd
```

#### Networking Information

```
/proc/net/
├── tcp                 # TCP connections
├── udp                 # UDP connections
├── arp                 # ARP table
└── route               # Routing table
```

#### Kernel Tuning

```
/proc/sys/
├── net/
│   ├── ipv4/
│   │   ├── ip_forward          # IP forwarding
│   │   └── tcp_syncookies      # SYN flood protection
├── vm/
│   └── swappiness              # Swap tendency
└── kernel/
    ├── hostname                # System hostname
    └── panic                   # Kernel panic behavior
```

**Example:**
```bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# View swappiness
cat /proc/sys/vm/swappiness
```

---

### /sys - System and Hardware Info

**Kernel interface for hardware and drivers (sysfs).**

Exposes:
- Hardware devices
- Drivers
- Kernel subsystems

❌ Not stored on disk (virtual)

#### Structure

```
/sys
├── block/              # Block devices
├── bus/                # Hardware buses
├── class/              # Device classes
├── devices/            # Physical device tree
├── firmware/           # Firmware interfaces
├── fs/                 # Filesystem info
├── kernel/             # Kernel parameters
├── module/             # Loaded modules
└── power/              # Power management
```

#### Block Devices

```
/sys/block/
├── sda/
│   ├── size
│   ├── queue/
│   └── sda1/
└── nvme0n1/
```

**Example:**
```bash
cat /sys/block/sda/size         # Disk size in sectors
```

#### Network Devices

```
/sys/class/net/
├── eth0/
│   ├── address                 # MAC address
│   ├── operstate               # Link state
│   └── speed                   # Link speed
└── lo/
```

**Example:**
```bash
cat /sys/class/net/eth0/address # MAC address
cat /sys/class/net/eth0/speed   # Link speed (Mbps)
```

#### CPU Control

```
/sys/devices/system/cpu/
├── cpu0/
├── cpu1/
├── online
└── offline
```

#### Kernel Modules

```
/sys/module/
├── ip_tables/
│   ├── parameters/
│   └── refcnt
└── nf_conntrack/
```

#### Power Management

```bash
cat /sys/power/state            # Available power states
echo mem > /sys/power/state     # Suspend to RAM (needs root)
```

---

## Runtime Data

### /run - Runtime Data

**Stores temporary runtime data since last boot.**

- Created at boot
- Lives in RAM
- Cleared on reboot
- Replaces legacy `/var/run`

#### Structure

```
/run
├── systemd/                    # Systemd runtime state
│   ├── system/
│   ├── user/
│   └── sessions/
│
├── lock/                       # Lock files
│
├── user/                       # User sessions
│   └── 1000/
│       ├── bus                 # D-Bus socket
│       └── systemd/
│
├── log/                        # Temporary logs
├── mount/                      # Mount-related data
│
├── sshd.pid                    # SSH daemon PID
├── docker.pid                  # Docker daemon PID
└── nginx.pid                   # Nginx PID
```

#### PID Files

Used to:
- Track running services
- Prevent duplicate starts
- Manage restarts

**Example:**
```bash
cat /run/sshd.pid
# 1234
```

#### User Sessions

Created at login, removed at logout:

```
/run/user/1000/
├── bus                         # D-Bus socket
├── systemd/                    # User systemd
└── gvfs/                       # Virtual filesystem
```

---

## Temporary Files

### /tmp - Temporary Files

**Temporary files cleared on reboot.**

✅ World-writable
✅ Cleared automatically (often on boot)
⚠️ Not secure (other users can see files)

#### Who Uses /tmp?

- Shell scripts
- Package installers
- Editors (vim, nano)
- Browsers
- Compilers
- System services

#### Structure

```
/tmp
├── tmp.abc123                  # Random temp files
├── nginx_temp/
├── systemd-private-xyz/
├── vim-root/
├── ssh-XXXXXX/
└── myscript.tmp
```

Names are usually randomized for security.

#### Cleanup

**Automatic cleanup:**
```bash
# systemd cleans /tmp based on age
cat /usr/lib/tmpfiles.d/tmp.conf

# Manual cleanup
sudo rm -rf /tmp/*              # ⚠️ Risky if services are running
```

---

## User System Resources

### /usr - User Programs and Data

**User system resources, applications, and libraries.**

- Read-only during normal operation
- Shared across all users
- OS-managed content
- Package manager controlled

❌ Don't modify manually

#### Structure

```
/usr
├── bin/                # User commands
├── sbin/               # System admin commands
├── lib/                # Libraries
├── lib64/              # 64-bit libraries
├── share/              # Architecture-independent data
├── local/              # Locally installed software
├── include/            # Header files
└── src/                # Source code
```

#### /usr/bin - User Commands

Most commands we use daily:

```bash
ls, cp, grep, awk, sed
python3, git, curl
vim, nano, emacs
```

#### /usr/sbin - System Admin Commands

Admin-only tools:

```bash
iptables, useradd, nginx
cron, mount, systemctl
```

#### /usr/lib and /usr/lib64 - Libraries

Non-essential libraries (boot doesn't depend on these):

```
/usr/lib
├── systemd/
├── python3/
├── openssh/
└── modules/
```

#### /usr/share - Shared Data

Architecture-independent data:

```
/usr/share
├── man/                # Manual pages
├── doc/                # Documentation
├── zoneinfo/           # Timezone data
├── icons/              # Icons
├── fonts/              # Fonts
└── applications/       # Desktop entries
```

#### /usr/include - Header Files

Used when compiling software:

```bash
/usr/include
├── stdio.h
├── stdlib.h
├── linux/
└── sys/
```

#### /usr/src - Source Code

Kernel and driver source:

```bash
/usr/src
├── linux-headers-6.5.0
└── kernel-source/
```

#### /usr/local - Locally Installed Software

**For manually compiled or installed software:**

```
/usr/local
├── bin/                # Local executables
├── sbin/               # Local admin tools
├── lib/                # Local libraries
├── share/              # Local shared data
└── etc/                # Local configs
```

**Used for:**
- Compiled-from-source tools
- Local scripts
- Admin-installed binaries

#### Modern Merging

On modern systems:
```bash
/bin → /usr/bin
/sbin → /usr/sbin
/lib → /usr/lib
```

Single unified `/usr` layout.

---

## Variable Data

### /var - Variable Runtime Data

**Variable data that grows over time and changes frequently.**

Contains:
- Logs
- Caches
- Queues
- Spool files
- Databases (sometimes)

#### Structure

```
/var
├── log/                # System and service logs
├── lib/                # Application state data
├── cache/              # Cached data
├── spool/              # Queued jobs
├── tmp/                # Persistent temp files
├── run/                # → /run (symlink)
├── mail/               # Local mail
├── opt/                # Runtime data for /opt apps
└── www/                # Web server root
```

#### /var/log - Logs

System and service logs:

```
/var/log
├── syslog              # System messages
├── messages            # General messages (RHEL)
├── kern.log            # Kernel messages
├── auth.log            # Authentication logs
├── boot.log            # Boot logs
├── dmesg               # Kernel ring buffer
│
├── journal/            # systemd journal
│   └── <machine-id>/
│
├── nginx/
│   ├── access.log
│   └── error.log
│
├── apache2/
├── mysql/
└── docker/
```

**Common log files:**
```bash
tail -f /var/log/syslog         # Follow system log
tail -f /var/log/auth.log       # Follow auth log
journalctl -f                   # Follow systemd journal
```

#### /var/lib - Application State

```
/var/lib
├── docker/             # Docker data
├── mysql/              # MySQL databases
├── postgresql/         # PostgreSQL databases
├── systemd/            # Systemd state
├── rpm/                # RPM database
├── dpkg/               # Debian package database
└── kubelet/            # Kubernetes data
```

#### /var/cache - Cached Data

Usually safe to clean:

```
/var/cache
├── apt/                # APT package cache
├── yum/                # YUM package cache
├── man/                # Man page cache
└── fontconfig/         # Font cache
```

**Cleanup:**
```bash
sudo apt clean                  # Clear APT cache
sudo yum clean all              # Clear YUM cache
```

#### /var/spool - Queued Jobs

Used for delayed processing:

```
/var/spool
├── cron/               # Cron job queue
├── mail/               # Mail queue
├── cups/               # Print queue
└── at/                 # at job queue
```

#### /var/tmp - Persistent Temp

**Unlike `/tmp`, survives reboot.**

```bash
/var/tmp                # Persistent temp files
```

#### /var/www - Web Server Root

Commonly used by Apache and Nginx:

```
/var/www
└── html/
    ├── index.html
    └── app/
```

#### /var/mail - Mail Storage

Local user mail:

```bash
/var/mail                       # User mailboxes
/var/spool/mail                 # Alternative location
```

---

## Service Data

### /srv - Service Data

**Data served by system services (FHS-defined).**

Contains:
- ✅ Service content
- ❌ Not binaries
- ❌ Not configs
- ❌ Not logs

#### What is "Served Data"?

Data that:
- Clients request
- Services expose
- Users download/access

#### Structure

```
/srv
├── www/                # Web content (HTML, static files)
├── ftp/                # FTP uploads/downloads
├── git/                # Git repositories
├── nfs/                # NFS exported directories
├── tftp/               # TFTP boot files
└── api/                # API data
```

#### Example: Web Server Using /srv/www

```
/srv/www/myapp
├── index.html
├── assets/
│   ├── css/
│   ├── js/
│   └── images/
└── uploads/
```

**Configured in nginx:**
```nginx
# /etc/nginx/nginx.conf
server {
    root /srv/www/myapp;
}
```

**Logs go to:**
```bash
/var/log/nginx/
```

---

## Quick Reference Tables

### Directory Purpose Summary

| Directory | Purpose | Type | Size |
|-----------|---------|------|------|
| `/bin` | Essential user binaries | Static | Small |
| `/boot` | Boot files, kernels | Static | Medium |
| `/dev` | Device files | Virtual | N/A |
| `/etc` | System configuration | Static | Small |
| `/home` | User home directories | Dynamic | Large |
| `/lib` | Shared libraries | Static | Medium |
| `/media` | Removable media mounts | Dynamic | N/A |
| `/mnt` | Temporary mounts | Dynamic | N/A |
| `/opt` | Optional software | Static | Medium |
| `/proc` | Process information | Virtual | N/A |
| `/root` | Root user home | Dynamic | Small |
| `/run` | Runtime data (RAM) | Dynamic | Small |
| `/sbin` | System binaries | Static | Small |
| `/srv` | Service data | Dynamic | Large |
| `/sys` | System/hardware info | Virtual | N/A |
| `/tmp` | Temporary files | Dynamic | Variable |
| `/usr` | User programs | Static | Large |
| `/var` | Variable data | Dynamic | Large |

### Filesystem Types

| Type | Examples | Characteristics |
|------|----------|----------------|
| **Real** | `/home`, `/var`, `/usr` | Stored on disk |
| **Virtual** | `/proc`, `/sys`, `/dev` | Lives in RAM |
| **Runtime** | `/run`, `/tmp` | Cleared on boot |

### Modern Symlinks

Many modern distributions use symlinks for compatibility:

```bash
/bin → /usr/bin
/sbin → /usr/sbin
/lib → /usr/lib
/var/run → /run
/var/lock → /run/lock
```

---

## Best Practices

### Directory Usage Guidelines

1. **Read-only directories** - Don't modify:
   - `/bin`, `/sbin`, `/lib`, `/usr`

2. **User data** - Normal users:
   - `/home/<username>`

3. **Admin work** - Root user:
   - `/root`

4. **Temporary work** - Use:
   - `/tmp` (cleared on boot)
   - `/var/tmp` (persistent)

5. **Custom apps** - Install to:
   - `/opt` (third-party)
   - `/usr/local` (compiled)

6. **Service data** - Store in:
   - `/srv` (served to clients)
   - `/var/lib` (application state)

### Disk Space Management

**Monitor these directories:**
```bash
du -sh /var/log                 # Log files
du -sh /var/cache               # Package cache
du -sh /home                    # User data
du -sh /tmp                     # Temp files

# Find largest directories
du -h /var | sort -rh | head -20
```

**Clean up space:**
```bash
# Clean package cache
sudo apt clean                  # Debian/Ubuntu
sudo yum clean all              # RHEL/CentOS

# Rotate logs
sudo logrotate -f /etc/logrotate.conf

# Clean old kernels
sudo apt autoremove             # Remove old kernels
```

### Security Considerations

**Protect sensitive directories:**
```bash
# Correct permissions
chmod 700 /root                 # Root home
chmod 700 ~/.ssh                # SSH config
chmod 600 ~/.ssh/id_rsa         # Private key

# Verify system files
rpm -Va                         # RHEL/CentOS
debsums -c                      # Debian/Ubuntu
```

**Monitor important files:**
```bash
# Check for unauthorized changes
sudo aide --check               # AIDE (Advanced Intrusion Detection)
```

---

This comprehensive guide covers the complete Linux directory structure following the Filesystem Hierarchy Standard (FHS).