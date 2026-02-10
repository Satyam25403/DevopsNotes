# User and Group Management

Comprehensive guide to managing users, groups, permissions, and sudo access in Linux.

## Table of Contents
- [Understanding Sudoers](#understanding-sudoers)
- [User Information and Monitoring](#user-information-and-monitoring)
- [User Management](#user-management)
- [Group Management](#group-management)
- [Best Practices](#best-practices)

---

## Understanding Sudoers

### What are Sudoers?

**Sudoers** are users allowed to run commands as another user (usually root) using `sudo`.

**Key Concepts:**
- They do NOT become root permanently
- They get temporary, controlled privilege escalation
- All sudo actions are logged for accountability

### Why Sudoers Exist

| Problem | Solution |
|---------|----------|
| Root has full power (dangerous) | Sudo provides controlled access |
| Giving root password to everyone ❌ | Each user keeps their own password |
| No accountability | All sudo commands are logged |
| Need least privilege | Grant specific permissions only |

### Sudo Configuration

**Main config file:** `/etc/sudoers`

⚠️ **Never edit directly!** Always use:
```bash
sudo visudo                            # Safe editor with syntax checking
```

**Example sudoers entry:**
```
john ALL=(ALL) ALL
```

**Format breakdown:**
```
user  host=(runas) commands
│     │    │       │
│     │    │       └─ Commands allowed
│     │    └───────── Can run as which user
│     └────────────── From which host
└──────────────────── Username
```

**Common patterns:**
```bash
# Full sudo access
john ALL=(ALL) ALL

# Specific command without password
jenkins ALL=(ALL) NOPASSWD:/usr/bin/systemctl restart nginx

# Multiple commands
admin ALL=(ALL) /usr/bin/systemctl, /usr/bin/journalctl

# Group access
%developers ALL=(ALL) ALL              # % indicates group
```

**Benefits of specific permissions:**
- ✅ Jenkins can only restart nginx
- ✅ No password required for automation
- ✅ No root shell access
- ✅ Security through least privilege

---

## User Information and Monitoring

### Current User Information

```bash
whoami                                 # Current logged-in username
id                                     # User ID, group ID, groups
id username                            # Info about specific user
```

**Example output:**
```
$ id
uid=1000(john) gid=1000(john) groups=1000(john),27(sudo),999(docker)
```

### Who is Logged In

```bash
who                                    # Users currently logged in
w                                      # Users and what they're running
w -f                                   # Include remote hosts (from where)
```

**Example output:**
```
$ w
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
john     pts/0    192.168.1.10     09:30    0.00s  0.04s  0.00s w
admin    pts/1    10.0.0.5         10:15    5:00   0.00s  0.00s bash
```

**Use cases:**
- Monitor active sessions
- Security auditing
- Incident analysis (who did what, when, from where)

### Login History

```bash
last                                   # Login history (who, when, from where)
last -n 10                             # Last 10 logins
last username                          # Login history for specific user
lastlog                                # Last login per user
lastlog -u username                    # Last login for specific user
```

**Example output:**
```
$ last
john     pts/0    192.168.1.10     Mon Jan 15 09:30   still logged in
admin    pts/1    10.0.0.5         Mon Jan 15 08:00 - 12:00  (04:00)
```

### User Switching

```bash
su - username                          # Switch to another user (needs their password)
sudo -i                                # Switch to root shell (needs your password)
exit                                   # Return to previous user
```

### Sudo Management

```bash
sudo -l                                # Show commands current user can run with sudo
sudo -k                                # Clear cached credentials (force password prompt)
sudo -v                                # Extend sudo timeout without running command
sudo ps -u username                    # Run command as another user
sudo -u username command               # Run command as specific user
```

**Example output:**
```
$ sudo -l
User john may run the following commands on server:
    (ALL) ALL
    (root) NOPASSWD: /usr/bin/systemctl restart nginx
```

---

## User Management

### Creating Users

⚠️ **Note:** Commands vary slightly by distribution:
- **Debian/Ubuntu:** `useradd`, `adduser`
- **RHEL/CentOS:** `useradd`

#### Basic User Creation

```bash
# Create user (minimal)
sudo useradd username

# User is added to private group with same name
# Example: user 'john' → group 'john'
```

#### User Creation with Home Directory

```bash
# Create user with home directory
sudo useradd -m username               # Creates /home/username
                                       # Sets ownership to user

# Verify
ls -ld /home/username
```

#### User Creation with Groups

```bash
# Create user with primary group
sudo useradd -m -g developers username

# Create user with secondary groups
sudo useradd -m -G docker,jenkins username

# Create user with primary AND secondary groups
sudo useradd -m -g developers -G docker,jenkins username
```

**Example:**
```bash
sudo useradd -m -g developers -G docker,jenkins alice
# Primary group: developers
# Secondary groups: docker, jenkins
# Home: /home/alice
```

#### Advanced User Creation

```bash
# Custom home directory
sudo useradd -m -d /opt/appuser username

# System/Service user (no login)
sudo useradd -r -s /sbin/nologin appuser
# Used for: nginx, jenkins, docker services
# Prevents: Shell access (security - attackers can't login)

# User with specific UID and GID
sudo useradd -u 1050 -g 1050 john

# User with specific shell
sudo useradd -m -s /bin/bash username

# Complete example
sudo useradd -m -s /bin/bash -g developers -G docker,sudo john
```

### Setting Passwords

```bash
# Set/update password for current user
passwd

# Set/update password for any user (requires sudo)
sudo passwd username

# Set password during user creation
sudo useradd -m username && sudo passwd username
```

⚠️ **Security Warning:**

**NEVER use `-p` flag with useradd:**
```bash
# WRONG - INSECURE!
sudo useradd -p "plaintext" username   # ❌ Expects HASHED password
                                       # ❌ Leaks via shell history
                                       # ❌ Visible in ps output
```

**CORRECT:**
```bash
# Always set password separately
sudo passwd username
```

### Viewing User Information

```bash
# Get user info
getent passwd username                 # Single user info

# View all users
cat /etc/passwd                        # All user accounts

# View hashed passwords and aging info
sudo cat /etc/shadow                   # Password hashes, expiry, aging

# Check user's groups
groups username
```

**Example /etc/passwd entry:**
```
john:x:1000:1000:John Doe:/home/john:/bin/bash
│    │ │    │    │         │          │
│    │ │    │    │         │          └─ Login shell
│    │ │    │    │         └─────────── Home directory
│    │ │    │    └───────────────────── Full name (GECOS)
│    │ │    └────────────────────────── Primary GID
│    │ └─────────────────────────────── UID
│    └───────────────────────────────── Password (x = in /etc/shadow)
└────────────────────────────────────── Username
```

### Modifying Users

```bash
# Lock user (disable login)
sudo usermod -L username               # Use case: After 3 failed attempts

# Unlock user (enable login)
sudo usermod -U username

# Add user to group (append)
sudo usermod -aG groupname username    # -a: append (don't remove from other groups)
                                       # -G: secondary group

# Change primary group
sudo usermod -g groupname username

# Make user a sudoer
sudo usermod -aG sudo username         # Debian/Ubuntu
sudo usermod -aG wheel username        # RHEL/CentOS

# Change user's shell
sudo usermod -s /bin/zsh username

# Change home directory
sudo usermod -d /new/home username

# Rename user
sudo usermod -l newname oldname
```

### Deleting Users

```bash
# Remove user (keep home directory)
sudo userdel username

# Remove user AND home directory
sudo userdel -r username               # ⚠️ Deletes /home/username

# Force remove (even if logged in)
sudo userdel -f username               # ⚠️ Dangerous!
```

### User Setup Verification

```bash
# Verify user creation
id username                            # User and group IDs
groups username                        # Group memberships
getent passwd username                 # Full user info
ls -ld /home/username                 # Home directory ownership
sudo -l -U username                    # Sudo permissions
```

**Complete verification example:**
```bash
$ id alice
uid=1001(alice) gid=1002(developers) groups=1002(developers),999(docker),27(sudo)

$ groups alice
alice : developers docker sudo

$ getent passwd alice
alice:x:1001:1002:Alice Smith:/home/alice:/bin/bash
```

---

## Group Management

### Understanding Groups

**Types of groups:**
1. **Primary group:** User's default group (one per user)
2. **Secondary groups:** Additional groups (multiple allowed)

**Common groups:**
- `sudo` / `wheel` - Sudoers
- `docker` - Docker access
- `www-data` - Web server
- `developers` - Development team

### Creating Groups

```bash
# Create group
sudo groupadd groupname

# Create group and add users
sudo groupadd developers -U alice,bob,charlie

# Create group with specific GID
sudo groupadd -g 1500 groupname
```

### Viewing Groups

```bash
# List all groups
cat /etc/group

# Get group info
getent group groupname                 # Group ID, members

# See user's groups
groups username
```

**Example /etc/group entry:**
```
developers:x:1002:alice,bob,charlie
│          │ │    │
│          │ │    └─ Members
│          │ └────── GID
│          └──────── Password (x = none)
└─────────────────── Group name
```

### Modifying Groups

```bash
# Rename group
sudo groupmod -n newname oldname

# Add users to group (append existing members)
sudo groupmod -aU alice,bob developers

# Change GID
sudo groupmod -g 2000 groupname
```

### Deleting Groups

```bash
# Delete group
sudo groupdel groupname

# Note: Cannot delete if it's a user's primary group
```

### Group Ownership and Permissions

```bash
# Check ownership
ls -l                                  # See file ownership

# Change group of file/directory
sudo chgrp groupname file
sudo chgrp -R groupname directory      # Recursive

# Group inheritance (SetGID)
chmod g+s directory                    # New files inherit directory's group
```

**SetGID example:**
```bash
# Setup shared directory
sudo mkdir /shared/project
sudo chgrp developers /shared/project
sudo chmod 2775 /shared/project        # 2 = SetGID bit
# OR
chmod g+s /shared/project

# Verify
ls -ld /shared/project
# drwxrws--- 2 root developers 4096 Jan 15 10:30 /shared/project
#      ↑
#   SetGID active
```

**Result:** Any file created in `/shared/project` automatically belongs to `developers` group.

---

## Best Practices

### User Management

1. **Use `-m` flag** when creating users to ensure home directory
   ```bash
   sudo useradd -m username              # Always create home
   ```

2. **Set password separately** - never use `-p` flag
   ```bash
   sudo useradd -m user && sudo passwd user
   ```

3. **Use service accounts** for applications
   ```bash
   sudo useradd -r -s /sbin/nologin nginx  # No shell = more secure
   ```

4. **Lock unused accounts** instead of deleting
   ```bash
   sudo usermod -L olduser               # Preserve data, prevent login
   ```

5. **Verify user setup** after creation
   ```bash
   id username
   groups username
   ls -ld /home/username
   ```

### Group Management

1. **Use groups for access control** instead of individual permissions
   ```bash
   # Better: Group-based
   sudo chgrp developers /project
   sudo chmod 770 /project
   
   # Worse: Individual users
   sudo chown user1 /project
   ```

2. **Use SetGID** for shared directories
   ```bash
   chmod g+s /shared/dir                 # Inherit group
   ```

3. **Name groups meaningfully**
   - Good: `developers`, `web-admins`, `db-users`
   - Bad: `group1`, `team`, `users`

4. **Document group purposes**
   ```bash
   # Comment in /etc/group or separate docs
   developers:x:1002:alice,bob    # Project team members
   ```

### Security

1. **Principle of least privilege**
   ```bash
   # Grant only necessary permissions
   jenkins ALL=(ALL) NOPASSWD:/usr/bin/systemctl restart nginx
   # NOT: jenkins ALL=(ALL) NOPASSWD:ALL
   ```

2. **Regular auditing**
   ```bash
   # Who has sudo access?
   getent group sudo
   
   # Review sudoers file
   sudo visudo -c                        # Check syntax
   ```

3. **Monitor login activity**
   ```bash
   last                                  # Login history
   w                                     # Current users
   journalctl -u ssh                     # SSH logs
   ```

4. **Use strong passwords**
   ```bash
   # Enforce password policies in /etc/login.defs
   PASS_MIN_LEN 12
   PASS_MAX_DAYS 90
   ```

5. **Disable root login via SSH**
   ```bash
   # In /etc/ssh/sshd_config
   PermitRootLogin no
   ```

### Organizational

1. **Naming conventions**
   - Users: `firstname.lastname` or `firstinitiallastname`
   - Service accounts: `app-servicename` (e.g., `app-nginx`)
   - Groups: `team-purpose` (e.g., `dev-backend`)

2. **Documentation**
   - Maintain list of users and their roles
   - Document group purposes
   - Record sudo permissions

3. **Automation**
   ```bash
   # Script for consistent user creation
   #!/bin/bash
   create_dev_user() {
       username=$1
       sudo useradd -m -s /bin/bash -G developers,docker "$username"
       sudo passwd "$username"
       echo "User $username created in developers group"
   }
   ```

---

## Common Workflows

### Onboarding New Developer

```bash
# 1. Create user with development groups
sudo useradd -m -s /bin/bash -g developers -G docker,sudo alice

# 2. Set password
sudo passwd alice

# 3. Setup SSH access (optional)
sudo mkdir -p /home/alice/.ssh
sudo cp authorized_keys /home/alice/.ssh/
sudo chown -R alice:alice /home/alice/.ssh
sudo chmod 700 /home/alice/.ssh
sudo chmod 600 /home/alice/.ssh/authorized_keys

# 4. Verify
id alice
groups alice
sudo -l -U alice
```

### Creating Service User

```bash
# 1. Create system user (no login)
sudo useradd -r -s /sbin/nologin -d /opt/myapp myapp

# 2. Create application directory
sudo mkdir -p /opt/myapp
sudo chown myapp:myapp /opt/myapp

# 3. Setup service
sudo systemctl edit myapp.service
# [Service]
# User=myapp
# Group=myapp
```

### Setting Up Shared Project Directory

```bash
# 1. Create group
sudo groupadd project-team

# 2. Add users to group
sudo usermod -aG project-team alice
sudo usermod -aG project-team bob

# 3. Create and configure directory
sudo mkdir /shared/project
sudo chgrp project-team /shared/project
sudo chmod 2775 /shared/project         # SetGID + rwxrwxr-x

# 4. Verify
ls -ld /shared/project
getent group project-team
```

---

## Troubleshooting

### User Can't Login

```bash
# Check if user is locked
sudo passwd -S username

# Check shell
getent passwd username | cut -d: -f7

# Check SSH settings
sudo grep "AllowUsers\|DenyUsers" /etc/ssh/sshd_config
```

### Permission Denied

```bash
# Check group membership
groups username

# Verify directory permissions
ls -ld /path/to/directory

# User may need to log out and back in for group changes to take effect
```

### Sudo Not Working

```bash
# Check sudo group membership
groups username | grep -E 'sudo|wheel'

# Verify sudoers file
sudo visudo -c

# Check sudo logs
sudo journalctl -u sudo
```

---

## Quick Reference

### User Commands

```bash
sudo useradd -m username               # Create user with home
sudo passwd username                   # Set password
sudo usermod -aG group username        # Add to group
sudo userdel -r username               # Delete user and home
id username                            # Show user info
groups username                        # Show user's groups
```

### Group Commands

```bash
sudo groupadd groupname                # Create group
sudo groupmod -n new old               # Rename group
sudo groupdel groupname                # Delete group
getent group groupname                 # Show group info
sudo chgrp group file                  # Change file group
chmod g+s directory                    # Set SetGID
```

### Monitoring Commands

```bash
whoami                                 # Current user
who                                    # Logged in users
w                                      # Users and activity
last                                   # Login history
sudo -l                                # My sudo permissions
```

---

This guide provides comprehensive coverage of user and group management for Linux system administration and DevOps operations.