# System and Host Information

A comprehensive guide to retrieving system, kernel, host, and distribution information in Linux.

## Table of Contents
- [System Information (Kernel-level)](#system-information-kernel-level)
- [Host Machine and Identity](#host-machine-and-identity)
- [Distribution Specific Info](#distribution-specific-info)
- [Package Management](#package-management)

---

## System Information (Kernel-level)

### uname - Unix Name
Display system information about the kernel and operating system.

```bash
uname           # Kernel name
uname -r        # Kernel version
uname -n        # Hostname of machine
uname -m        # Machine hardware architecture
uname -o        # Operating system
uname -p        # Processor type
uname -a        # All possible info about the system
man uname       # Manual page for uname
```

---

## Host Machine and Identity

### hostname
Shows complete host information including hostname, OS, kernel, architecture, and virtualization.

```bash
hostname -I     # IP address of machine
hostname -f     # Fully qualified domain name (FQDN)
man hostname    # Manual page for hostname
```

### hostnamectl
Modern systemd-based utility for host information and hostname management.

```bash
hostnamectl                          # Display all host information
sudo hostnamectl set-hostname <name> # Set new hostname
```

**Example output:**
```
Static hostname: myserver
Icon name: computer-vm
Chassis: vm
Machine ID: abc123...
Boot ID: xyz789...
Virtualization: kvm
Operating System: Ubuntu 22.04 LTS
Kernel: Linux 5.15.0-56-generic
Architecture: x86-64
```

---

## Distribution Specific Info

### lsb_release
Linux Standard Base release information.

```bash
lsb_release -i                    # Distribution ID (e.g., Ubuntu, Debian)
lsb_release -r                    # Release version
lsb_release -a                    # All available info
sudo apt install lsb-release      # Install if missing (Debian/Ubuntu)
```

### Other Information Commands

```bash
arch                    # Architecture info (e.g., x86_64)
cat /proc/version       # Detailed system info including kernel and gcc version
```

### Kernel Modules

```bash
lsmod                   # List all loaded kernel modules
modinfo <module>        # Display information about a specific module
man modinfo             # Manual page for modinfo
```

---

## Package Management

Linux distributions use different package managers. Choose commands based on your distribution.

### Debian/Ubuntu (APT - Advanced Package Tool)

```bash
sudo apt update                      # Refresh local package index from repositories
sudo apt upgrade                     # Upgrade all installed packages to latest versions
sudo apt install <packagename>       # Install a package
sudo apt remove <packagename>        # Remove a package
sudo apt search <packagename>        # Search for a package
sudo apt show <packagename>          # Show package details
```

**Important:** Always run `apt update` before `apt upgrade` or `apt install`

### RHEL/CentOS/Fedora (YUM)

```bash
sudo yum update                      # Update system packages
sudo yum install <packagename>       # Install a package
sudo yum remove <packagename>        # Remove a package
```

### Fedora/RHEL 8+ (DNF)
Modern replacement for YUM with improved performance.

```bash
sudo dnf update                      # Update system packages
sudo dnf install <packagename>       # Install a package
sudo dnf remove <packagename>        # Remove a package
```

### OpenSUSE (Zypper)

```bash
sudo zypper refresh                  # Refresh repositories
sudo zypper install <packagename>    # Install a package
sudo zypper remove <packagename>     # Remove a package
```

---

## Working with Package Files (Low-level)

### Debian Packages (dpkg)

Direct package file manipulation tool.

```bash
sudo dpkg -i <package-file>.deb      # Install .deb file
sudo dpkg -r <pkgname>               # Remove package
sudo dpkg -s <pkgname>               # Show package status
dpkg -L <pkgname>                    # List all files installed by package
dpkg -l                              # List all installed packages
```

⚠️ **Note:** `dpkg` does not resolve dependencies automatically. Use `apt` for dependency management.

### RHEL Packages (RPM)

```bash
sudo rpm -ivh <package>.rpm          # Install RPM package (-i install, -v verbose, -h hash marks)
sudo rpm -e <pkgname>                # Erase/remove RPM package
rpm -qa                              # Query all installed RPM packages
rpm -qi <pkgname>                    # Query info about installed package
rpm -ql <pkgname>                    # Query list of files from package
```

---

## Universal Package Manager (SNAP)

Snap works across almost all Linux distributions with containerized packages.

```bash
sudo snap install <packagename>      # Install snap package
sudo snap remove <packagename>       # Remove snap package
snap list                            # List installed snap packages
snap info <pkgname>                  # Detailed info about snap package
sudo snap refresh                    # Update all installed snap packages
```

---

## Quick Reference Table

| Distribution | Package Manager | Update Repos | Install Package | Remove Package |
|--------------|----------------|--------------|-----------------|----------------|
| Debian/Ubuntu | APT | `apt update` | `apt install` | `apt remove` |
| RHEL/CentOS | YUM | `yum update` | `yum install` | `yum remove` |
| Fedora/RHEL 8+ | DNF | `dnf update` | `dnf install` | `dnf remove` |
| OpenSUSE | Zypper | `zypper refresh` | `zypper install` | `zypper remove` |
| Universal | Snap | `snap refresh` | `snap install` | `snap remove` |

---

## Best Practices

1. **Always update package index** before installing packages
2. **Use high-level package managers** (apt, yum, dnf) instead of low-level ones (dpkg, rpm) when possible
3. **Regularly update** your system for security patches
4. **Check package details** with `show` or `info` commands before installing
5. **Keep track of installed packages** for system maintenance and troubleshooting