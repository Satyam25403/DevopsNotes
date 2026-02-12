# Remote Access Protocols and Tools

Comprehensive guide to remote access, file transfer, and secure communication in Linux systems.

## Table of Contents
- [SSH - Secure Shell](#ssh---secure-shell)
- [SCP - Secure Copy](#scp---secure-copy)
- [SFTP - SSH File Transfer Protocol](#sftp---ssh-file-transfer-protocol)
- [rsync - Remote Sync](#rsync---remote-sync)
- [Telnet - Insecure Remote Access](#telnet---insecure-remote-access)
- [FTP - File Transfer Protocol](#ftp---file-transfer-protocol)
- [VNC - Virtual Network Computing](#vnc---virtual-network-computing)
- [RDP - Remote Desktop Protocol](#rdp---remote-desktop-protocol)
- [Security Best Practices](#security-best-practices)

---

## SSH - Secure Shell

**Secure, encrypted remote access protocol for managing Linux servers.**

### Overview

**SSH (Secure Shell):**
- Port: **22** (TCP)
- Encryption: Yes ✅
- Authentication: Password, Public Key, Certificate
- Use cases: Remote shell, file transfer, tunneling

**Advantages:**
- ✅ Encrypted communication
- ✅ Strong authentication
- ✅ Port forwarding
- ✅ File transfer (SCP, SFTP)
- ✅ X11 forwarding (GUI apps)

### Installation

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install openssh-server openssh-client

# RHEL/CentOS
sudo yum install openssh-server openssh-clients

# Start and enable SSH service
sudo systemctl start sshd
sudo systemctl enable sshd

# Check status
sudo systemctl status sshd
```

### Basic Usage

#### Connecting to Remote Server

```bash
# Basic connection
ssh username@hostname
ssh username@192.168.1.100

# Specify port (if not default 22)
ssh -p 2222 username@hostname

# Specify user (if different from current)
ssh user@server

# Examples
ssh alice@server.example.com
ssh root@192.168.1.50
ssh -p 2222 admin@10.0.0.5
```

#### First Connection

```bash
ssh alice@server.com

# Output:
The authenticity of host 'server.com (192.168.1.100)' can't be established.
ECDSA key fingerprint is SHA256:abc123...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes

# Host added to ~/.ssh/known_hosts
```

### SSH Key-Based Authentication

**More secure than passwords - recommended for all servers.**

#### Generate SSH Key Pair

```bash
# Generate ED25519 key (recommended)
ssh-keygen -t ed25519 -C "your_email@example.com"

# Generate RSA key (legacy compatibility)
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Output:
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/user/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/user/.ssh/id_ed25519
Your public key has been saved in /home/user/.ssh/id_ed25519.pub
```

**Key types:**
- **ED25519** - Modern, fast, secure (recommended)
- **RSA** - Widely compatible, use 4096 bits minimum
- **ECDSA** - Good, but ED25519 preferred
- **DSA** - Deprecated, insecure

#### Copy Public Key to Server

**Method 1: Using ssh-copy-id (easiest)**
```bash
ssh-copy-id username@server

# Specify key file
ssh-copy-id -i ~/.ssh/id_ed25519.pub username@server

# Specify port
ssh-copy-id -p 2222 username@server
```

**Method 2: Manual copy**
```bash
# Copy public key to clipboard
cat ~/.ssh/id_ed25519.pub

# On server, add to authorized_keys
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "ssh-ed25519 AAAAC3Nza... user@host" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

**Method 3: One-liner**
```bash
cat ~/.ssh/id_ed25519.pub | ssh username@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

#### Test Key-Based Authentication

```bash
ssh username@server
# Should login without password prompt

# Verbose mode to debug
ssh -v username@server
```

### SSH Configuration

#### Client Configuration (~/.ssh/config)

**Create convenient aliases and defaults:**

```bash
# ~/.ssh/config
Host myserver
    HostName 192.168.1.100
    User alice
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host prod
    HostName production.example.com
    User admin
    Port 2222
    IdentityFile ~/.ssh/prod_key
    ForwardAgent yes

Host *.example.com
    User deploy
    IdentityFile ~/.ssh/deploy_key
    StrictHostKeyChecking no

# Usage:
ssh myserver       # Instead of: ssh alice@192.168.1.100
ssh prod           # Instead of: ssh -p 2222 admin@production.example.com
```

**Common options:**
```
HostName            # Server address
User                # Username
Port                # SSH port
IdentityFile        # Private key path
ForwardAgent        # Agent forwarding
ForwardX11          # X11 forwarding
StrictHostKeyChecking  # Host key verification
ServerAliveInterval    # Keep-alive interval
```

#### Server Configuration (/etc/ssh/sshd_config)

**Secure SSH server configuration:**

```bash
# /etc/ssh/sshd_config

# Change default port (security through obscurity)
Port 2222

# Disable root login (security best practice)
PermitRootLogin no

# Allow specific users only
AllowUsers alice bob admin

# Disable password authentication (key-only)
PasswordAuthentication no
PubkeyAuthentication yes

# Disable empty passwords
PermitEmptyPasswords no

# Limit authentication attempts
MaxAuthTries 3

# Session timeouts
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable X11 forwarding (if not needed)
X11Forwarding no

# Enable strict mode
StrictModes yes

# Log level
LogLevel VERBOSE

# Apply changes
sudo systemctl restart sshd
```

**Test configuration before restarting:**
```bash
sudo sshd -t                    # Test config syntax
sudo sshd -T                    # Show full config
```

### Advanced SSH Features

#### SSH Tunneling (Port Forwarding)

**Local Port Forwarding:**

Forward local port to remote destination.

```bash
# Forward local port 8080 to remote server's port 80
ssh -L 8080:localhost:80 user@server

# Access remote database locally
ssh -L 3306:localhost:3306 user@dbserver
# Then: mysql -h 127.0.0.1 -P 3306

# Access internal service through jump host
ssh -L 9000:internal-server:80 user@jumphost
# Then browse: http://localhost:9000
```

**Remote Port Forwarding:**

Forward remote port to local destination.

```bash
# Allow remote server to access local service
ssh -R 8080:localhost:80 user@server
# Server can access: http://localhost:8080

# Expose local web app to remote server
ssh -R 3000:localhost:3000 user@server
```

**Dynamic Port Forwarding (SOCKS Proxy):**

```bash
# Create SOCKS proxy on port 1080
ssh -D 1080 user@server

# Configure browser to use localhost:1080 as SOCKS proxy
# All traffic goes through SSH tunnel
```

#### SSH Agent Forwarding

**Allows using local SSH keys on remote servers:**

```bash
# Enable agent forwarding
ssh -A user@server

# In ~/.ssh/config
Host server
    ForwardAgent yes

# Use case: Jump through bastion to internal server
ssh -A user@bastion
# Then from bastion:
ssh internal-server  # Uses your local key
```

#### ProxyJump (Jump Host)

**Connect through intermediate server:**

```bash
# Connect through bastion
ssh -J bastion-user@bastion.com user@internal-server

# Multiple jumps
ssh -J jump1,jump2 user@final-server

# In ~/.ssh/config
Host internal
    HostName 10.0.0.50
    User alice
    ProxyJump bastion.example.com
```

#### X11 Forwarding

**Run GUI applications remotely:**

```bash
# Enable X11 forwarding
ssh -X user@server

# Run GUI app
firefox &
gedit &

# Trusted X11 forwarding (faster, less secure)
ssh -Y user@server
```

**Server config requirement:**
```bash
# /etc/ssh/sshd_config
X11Forwarding yes
X11DisplayOffset 10
```

#### SSH Multiplexing

**Reuse connections for faster subsequent logins:**

```bash
# ~/.ssh/config
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600

# Create socket directory
mkdir -p ~/.ssh/sockets

# First connection creates master
ssh server

# Subsequent connections reuse master (instant)
ssh server
scp file server:
```

### SSH Key Management

#### Managing Multiple Keys

```bash
# Generate different keys for different purposes
ssh-keygen -t ed25519 -f ~/.ssh/id_work -C "work@example.com"
ssh-keygen -t ed25519 -f ~/.ssh/id_personal -C "personal@example.com"

# Use specific key
ssh -i ~/.ssh/id_work user@work-server
ssh -i ~/.ssh/id_personal user@personal-server

# In ~/.ssh/config
Host work
    IdentityFile ~/.ssh/id_work

Host personal
    IdentityFile ~/.ssh/id_personal
```

#### SSH Agent

**Manage keys in memory (avoid password re-entry):**

```bash
# Start SSH agent
eval $(ssh-agent)

# Add key to agent
ssh-add ~/.ssh/id_ed25519
# Enter passphrase once

# Add with timeout (hours)
ssh-add -t 8h ~/.ssh/id_ed25519

# List loaded keys
ssh-add -l

# Remove all keys
ssh-add -D

# Auto-start on login (add to ~/.bashrc)
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval $(ssh-agent)
    ssh-add ~/.ssh/id_ed25519
fi
```

#### Key Security

```bash
# Correct permissions (critical!)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/config

# Verify
ls -la ~/.ssh
```

### Troubleshooting SSH

#### Common Issues

**Permission denied:**
```bash
# Verbose output
ssh -v user@server
ssh -vv user@server         # More verbose
ssh -vvv user@server        # Maximum verbosity

# Check key permissions
ls -la ~/.ssh/id_ed25519    # Should be 600

# Verify key is being used
ssh-add -l                  # List loaded keys

# Test authentication methods
ssh -o PreferredAuthentications=publickey user@server
ssh -o PreferredAuthentications=password user@server
```

**Connection refused:**
```bash
# Check if SSH is running
sudo systemctl status sshd

# Check firewall
sudo ufw status
sudo iptables -L -n | grep 22

# Verify SSH is listening
sudo ss -tulnp | grep :22
sudo netstat -tulnp | grep :22
```

**Host key verification failed:**
```bash
# Remove old host key
ssh-keygen -R hostname
ssh-keygen -R 192.168.1.100

# Or edit known_hosts manually
nano ~/.ssh/known_hosts
```

#### Debug Server Side

```bash
# Check SSH logs
sudo tail -f /var/log/auth.log          # Debian/Ubuntu
sudo tail -f /var/log/secure            # RHEL/CentOS
sudo journalctl -u sshd -f              # systemd

# Test config
sudo sshd -t

# Run SSH in debug mode (as root)
sudo /usr/sbin/sshd -d -p 2222
```

---

## SCP - Secure Copy

**Secure file transfer over SSH.**

### Basic Usage

```bash
# Copy file to remote server
scp localfile.txt user@server:/remote/path/

# Copy file from remote server
scp user@server:/remote/file.txt /local/path/

# Copy directory recursively
scp -r localdir/ user@server:/remote/path/

# Specify port
scp -P 2222 file.txt user@server:/path/

# Preserve timestamps and permissions
scp -p file.txt user@server:/path/

# Verbose output
scp -v file.txt user@server:/path/
```

### Examples

```bash
# Upload file
scp report.pdf alice@192.168.1.100:/home/alice/documents/

# Download file
scp alice@192.168.1.100:/var/log/app.log /tmp/

# Upload directory
scp -r /local/project/ alice@server:/home/alice/projects/

# Copy between remote servers (via local)
scp user@server1:/path/file user@server2:/path/

# Copy with compression
scp -C largefile.tar.gz user@server:/backups/

# Limit bandwidth (KB/s)
scp -l 1000 file.txt user@server:/path/
```

### SCP with SSH Config

```bash
# ~/.ssh/config
Host myserver
    HostName 192.168.1.100
    User alice

# Then simply:
scp file.txt myserver:/path/
scp myserver:/path/file.txt ./
```

---

## SFTP - SSH File Transfer Protocol

**Interactive file transfer over SSH.**

### Basic Usage

```bash
# Connect to server
sftp user@server

# Specify port
sftp -P 2222 user@server
```

### SFTP Commands

**Once connected:**

```bash
sftp> help                      # List all commands

# Navigation
sftp> pwd                       # Print remote working directory
sftp> lpwd                      # Print local working directory
sftp> ls                        # List remote files
sftp> lls                       # List local files
sftp> cd /remote/path           # Change remote directory
sftp> lcd /local/path           # Change local directory

# Upload files
sftp> put localfile.txt         # Upload to current remote dir
sftp> put localfile.txt remotefile.txt
sftp> put -r localdir/          # Upload directory

# Download files
sftp> get remotefile.txt        # Download to current local dir
sftp> get remotefile.txt localfile.txt
sftp> get -r remotedir/         # Download directory

# File operations
sftp> mkdir newdir              # Create remote directory
sftp> rmdir dirname             # Remove remote directory
sftp> rm file.txt               # Delete remote file
sftp> rename old.txt new.txt    # Rename remote file
sftp> chmod 644 file.txt        # Change permissions

# Exit
sftp> exit
sftp> bye
sftp> quit
```

### Batch Mode

```bash
# Create batch file
cat > sftp-batch.txt << EOF
cd /remote/path
put file1.txt
put file2.txt
get remotefile.txt
exit
EOF

# Execute batch
sftp -b sftp-batch.txt user@server
```

---

## rsync - Remote Sync

**Efficient file synchronization and transfer.**

### Why rsync?

**Advantages over SCP:**
- ✅ Only transfers differences (faster for updates)
- ✅ Resume interrupted transfers
- ✅ Preserve permissions, timestamps, symlinks
- ✅ Compression
- ✅ Progress display
- ✅ Exclude patterns

### Installation

```bash
# Debian/Ubuntu
sudo apt install rsync

# RHEL/CentOS
sudo yum install rsync
```

### Basic Usage

```bash
# Copy local to local
rsync -avz source/ destination/

# Copy to remote server
rsync -avz source/ user@server:/remote/path/

# Copy from remote server
rsync -avz user@server:/remote/path/ /local/path/

# Sync directories (delete files in dest not in source)
rsync -avz --delete source/ user@server:/remote/path/
```

### Common Options

```bash
-a      # Archive mode (recursive, preserves permissions, times, etc.)
-v      # Verbose
-z      # Compression
-h      # Human-readable
-P      # Progress + partial (resume)
--delete    # Delete files in dest not in source
--exclude   # Exclude pattern
-n      # Dry run (test without making changes)
-e      # Specify remote shell
```

### Examples

```bash
# Backup with progress
rsync -avzP /home/user/ user@backup-server:/backups/user/

# Sync website
rsync -avz --delete /var/www/html/ user@webserver:/var/www/html/

# Backup excluding cache
rsync -avz --exclude 'cache/*' --exclude '*.tmp' \
    /data/ user@backup:/backups/

# Dry run (test)
rsync -avzn --delete source/ dest/

# Specify SSH port
rsync -avz -e "ssh -p 2222" source/ user@server:/path/

# Show progress
rsync -avzP largefile.tar.gz user@server:/backups/

# Limit bandwidth (KB/s)
rsync --bwlimit=1000 -avz source/ user@server:/path/

# Preserve hard links
rsync -aH source/ dest/
```

### Automated Backups

```bash
#!/bin/bash
# backup.sh

SOURCE="/home/user/documents"
DEST="user@backup-server:/backups/documents"
LOG="/var/log/backup.log"

rsync -avz --delete \
    --exclude '.cache/*' \
    --exclude 'Downloads/*' \
    "$SOURCE/" "$DEST" >> "$LOG" 2>&1

# Add to cron
# 0 2 * * * /path/to/backup.sh
```

---

## Telnet - Insecure Remote Access

**⚠️ DEPRECATED - Use SSH instead!**

### Overview

**Telnet:**
- Port: **23** (TCP)
- Encryption: **NO** ❌
- Status: Deprecated, insecure

**Why avoid Telnet:**
- ❌ No encryption (passwords sent in plaintext)
- ❌ Vulnerable to eavesdropping
- ❌ No authentication verification
- ❌ Superseded by SSH

### Limited Use Cases

**Only use Telnet for:**
- Testing port connectivity
- Debugging network services
- Accessing legacy devices (switches, routers)

### Installation

```bash
# Debian/Ubuntu
sudo apt install telnet

# RHEL/CentOS
sudo yum install telnet
```

### Usage

```bash
# Test port connectivity
telnet hostname 80
telnet 192.168.1.100 22

# Example: Test HTTP
telnet google.com 80
GET / HTTP/1.1
Host: google.com
[Enter twice]

# Exit
Ctrl+]
telnet> quit
```

**⚠️ Never use for actual remote access - use SSH instead!**

---

## FTP - File Transfer Protocol

**File transfer protocol (insecure - use SFTP instead).**

### Overview

**FTP:**
- Ports: **20** (data), **21** (control)
- Encryption: No (use FTPS or SFTP)
- Modes: Active, Passive

**Better alternatives:**
- ✅ SFTP (SSH File Transfer)
- ✅ FTPS (FTP over TLS)

### FTP Client Usage

```bash
# Install FTP client
sudo apt install ftp

# Connect
ftp hostname
ftp 192.168.1.100

# Login
Name: username
Password: ******

# Commands
ftp> ls                         # List files
ftp> cd directory               # Change directory
ftp> get file.txt               # Download file
ftp> put file.txt               # Upload file
ftp> mget *.txt                 # Download multiple
ftp> mput *.txt                 # Upload multiple
ftp> binary                     # Binary mode (for images, etc.)
ftp> ascii                      # ASCII mode (for text)
ftp> bye                        # Exit
```

### FTP Server (vsftpd)

**If you must use FTP (prefer SFTP):**

```bash
# Install vsftpd
sudo apt install vsftpd

# Configure
sudo nano /etc/vsftpd.conf

# Basic config
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES

# Restart
sudo systemctl restart vsftpd
```

**⚠️ Security Warning:**
- FTP transmits credentials in plaintext
- Use SFTP (over SSH) instead for security

---

## VNC - Virtual Network Computing

**Remote desktop access (graphical).**

### Overview

**VNC:**
- Port: **5900+** (5901, 5902, etc.)
- Protocol: RFB (Remote Framebuffer)
- Use case: Remote GUI access

### Server Setup (x11vnc)

```bash
# Install x11vnc
sudo apt install x11vnc

# Set password
x11vnc -storepasswd

# Start server
x11vnc -display :0 -auth guess -forever -loop -noxdamage \
    -repeat -rfbauth ~/.vnc/passwd -rfbport 5900 -shared

# Start on boot
# Create systemd service: /etc/systemd/system/x11vnc.service
[Unit]
Description=x11vnc
After=display-manager.service

[Service]
ExecStart=/usr/bin/x11vnc -display :0 -auth guess -forever
Restart=always

[Install]
WantedBy=multi-user.target
```

### VNC Client

```bash
# Install vncviewer
sudo apt install xtightvncviewer

# Connect
vncviewer 192.168.1.100:5900

# Or use GUI clients
# - TigerVNC
# - RealVNC
# - TightVNC
```

### SSH Tunnel for VNC

**Secure VNC over SSH:**

```bash
# Create SSH tunnel
ssh -L 5900:localhost:5900 user@server

# Connect VNC to localhost
vncviewer localhost:5900

# All VNC traffic encrypted through SSH
```

---

## RDP - Remote Desktop Protocol

**Windows remote desktop (can access from Linux).**

### Overview

**RDP:**
- Port: **3389** (TCP)
- Protocol: Remote Desktop Protocol
- Use case: Access Windows desktops

### Linux RDP Client (rdesktop)

```bash
# Install rdesktop
sudo apt install rdesktop

# Connect
rdesktop 192.168.1.100

# Full screen
rdesktop -f 192.168.1.100

# Specific resolution
rdesktop -g 1920x1080 192.168.1.100

# With sound
rdesktop -r sound:local 192.168.1.100
```

### Linux RDP Client (FreeRDP)

**More modern RDP client:**

```bash
# Install
sudo apt install freerdp2-x11

# Connect
xfreerdp /v:192.168.1.100 /u:username

# Full screen
xfreerdp /v:192.168.1.100 /u:username /f

# With folder sharing
xfreerdp /v:192.168.1.100 /u:username /drive:share,/home/user/shared
```

### Linux RDP Server (xrdp)

**Allow RDP access to Linux:**

```bash
# Install
sudo apt install xrdp

# Start service
sudo systemctl start xrdp
sudo systemctl enable xrdp

# Check status
sudo systemctl status xrdp

# Allow through firewall
sudo ufw allow 3389/tcp

# Connect from Windows
# Use Remote Desktop Connection
# Enter: linux-server-ip:3389
```

---

## Security Best Practices

### SSH Security Hardening

**Essential security measures:**

```bash
# 1. Disable root login
PermitRootLogin no

# 2. Use key-based authentication only
PasswordAuthentication no
PubkeyAuthentication yes

# 3. Change default port
Port 2222

# 4. Limit user access
AllowUsers alice bob
DenyUsers baduser

# 5. Set idle timeout
ClientAliveInterval 300
ClientAliveCountMax 2

# 6. Disable empty passwords
PermitEmptyPasswords no

# 7. Use strong ciphers
Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512,hmac-sha2-256

# 8. Enable fail2ban
sudo apt install fail2ban
```

### Firewall Configuration

```bash
# Allow SSH (custom port)
sudo ufw allow 2222/tcp

# Limit SSH attempts
sudo ufw limit 2222/tcp

# Deny old SSH port
sudo ufw deny 22/tcp

# For VNC over SSH only
# Don't open VNC port externally
sudo ufw deny 5900/tcp
```

### Key Management

**Best practices:**

```bash
# 1. Use strong keys
ssh-keygen -t ed25519 -a 100

# 2. Use passphrases
# Always protect private keys with passphrase

# 3. Rotate keys regularly
# Generate new keys yearly

# 4. Separate keys per use
# Work, personal, server-specific

# 5. Backup keys securely
# Encrypted backup of private keys

# 6. Revoke compromised keys
# Remove from authorized_keys immediately
```

### Monitoring and Logging

```bash
# Monitor SSH login attempts
sudo tail -f /var/log/auth.log | grep sshd

# Failed login attempts
sudo grep "Failed password" /var/log/auth.log

# Successful logins
sudo grep "Accepted publickey" /var/log/auth.log

# Install fail2ban
sudo apt install fail2ban

# Configure fail2ban for SSH
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 2222
maxretry = 3
bantime = 3600
```

---

## Quick Reference

### SSH Commands

```bash
ssh user@host                           # Basic connection
ssh -p 2222 user@host                   # Custom port
ssh -i ~/.ssh/key user@host             # Specific key
ssh -L 8080:localhost:80 user@host      # Local forward
ssh -R 8080:localhost:80 user@host      # Remote forward
ssh -D 1080 user@host                   # SOCKS proxy
ssh -J jump user@final                  # Jump host
```

### File Transfer

```bash
scp file.txt user@host:/path/           # Copy file
scp -r dir/ user@host:/path/            # Copy directory
sftp user@host                          # Interactive transfer
rsync -avzP src/ user@host:/dst/        # Efficient sync
```

### Key Management

```bash
ssh-keygen -t ed25519                   # Generate key
ssh-copy-id user@host                   # Copy key to server
ssh-add ~/.ssh/id_ed25519               # Add to agent
ssh-add -l                              # List keys
```

---

This comprehensive guide covers all major remote access protocols and secure file transfer methods for Linux systems.