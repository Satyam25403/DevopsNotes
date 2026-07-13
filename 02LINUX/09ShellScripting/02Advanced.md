# Advanced Shell Scripting

Professional-grade shell scripting techniques for production systems, DevOps, and infrastructure automation.

## Table of Contents
- [Advanced File Handling](#advanced-file-handling)
- [Deep Error Handling](#deep-error-handling)
- [Signals and Traps (Real Cases)](#signals-and-traps-real-cases)
- [Defensive Scripting Patterns](#defensive-scripting-patterns)
- [Idempotency](#idempotency)
- [Debugging Techniques](#debugging-techniques)
- [Production-Ready Script Example](#production-ready-script-example)
- [Interview One-Liners](#interview-one-liners)

---

## Advanced File Handling

### File Comparison Operators

**Beyond basic `-f` and `-d`:**

```bash
#!/bin/bash

# File age comparison
if [ file1 -nt file2 ]; then
    echo "file1 is newer than file2"
fi

if [ file1 -ot file2 ]; then
    echo "file1 is older than file2"
fi

# Same file (hard links)
if [ file1 -ef file2 ]; then
    echo "file1 and file2 are the same file"
fi
```

**Use case: Incremental backups**
```bash
#!/bin/bash

SOURCE="/data/important.txt"
BACKUP="/backup/important.txt"

if [ "$SOURCE" -nt "$BACKUP" ] || [ ! -f "$BACKUP" ]; then
    echo "Source has changed, creating backup..."
    cp "$SOURCE" "$BACKUP"
else
    echo "Backup is up to date"
fi
```

### Ownership and Permission Checks

```bash
#!/bin/bash

FILE="/etc/nginx/nginx.conf"

# Check if owned by current user
if [ -O "$FILE" ]; then
    echo "You own this file"
fi

# Check if group-owned
if [ -G "$FILE" ]; then
    echo "Your group owns this file"
fi

# Combined checks
if [ -O "$FILE" ] && [ -w "$FILE" ]; then
    echo "You can modify this file"
fi
```

### Symbolic Link Checks

```bash
#!/bin/bash

# Check if symbolic link
if [ -L /etc/nginx/nginx.conf ]; then
    echo "This is a symbolic link"
    
    # Get target
    target=$(readlink /etc/nginx/nginx.conf)
    echo "Points to: $target"
    
    # Get absolute path
    real_path=$(readlink -f /etc/nginx/nginx.conf)
    echo "Real path: $real_path"
fi
```

**Important for:**
- `/etc/alternatives` system
- `/var/log` symlinks
- Docker volumes
- Application deployments

### Safe File Creation (Avoid Overwrite)

**Prevent accidental overwrites:**

```bash
#!/bin/bash

# Enable noclobber
set -o noclobber

# This fails if file exists
echo "data" > file.txt  # Error if file exists

# Force overwrite if needed
echo "data" >| file.txt  # Works even with noclobber

# Disable noclobber
set +o noclobber
```

**Check before creating:**
```bash
#!/bin/bash

FILE="config.conf"

if [ -f "$FILE" ]; then
    read -p "File exists. Overwrite? (y/n) " confirm
    if [ "$confirm" != "y" ]; then
        echo "Aborting"
        exit 1
    fi
fi

echo "data" > "$FILE"
```

### Atomic File Write (CRITICAL)

**❌ Wrong way (can corrupt file):**
```bash
#!/bin/bash

# Never do this!
echo "new data" > config.conf  # File can be corrupted if script crashes
```

**✅ Correct way (atomic write):**
```bash
#!/bin/bash

# Create temp file
tmpfile=$(mktemp)

# Write to temp file
echo "new data" > "$tmpfile"

# Atomically move (replace)
mv "$tmpfile" config.conf

# mv is atomic on same filesystem
```

**Production-grade atomic write:**
```bash
#!/bin/bash

atomic_write() {
    local file=$1
    local content=$2
    
    # Create temp file in same directory (same filesystem)
    local tmpfile="${file}.tmp.$$"
    
    # Write content
    echo "$content" > "$tmpfile"
    
    # Set same permissions as original
    if [ -f "$file" ]; then
        chmod --reference="$file" "$tmpfile"
        chown --reference="$file" "$tmpfile"
    fi
    
    # Atomic replace
    mv "$tmpfile" "$file"
}

# Usage
atomic_write "/etc/myapp/config.conf" "server=localhost"
```

**Used in:**
- Config updates
- Deployments
- Secrets rotation
- Database migrations
- Container configs

---

## Deep Error Handling

### The Truth About `set -e`

**Common misconception:**
> "`set -e` exits on any error"

**❌ This is NOT always true!**

### When `set -e` Does NOT Exit

**1. In conditionals:**
```bash
#!/bin/bash
set -e

if false; then
    echo "won't run"
fi

echo "still runs"  # This executes!
```

**2. With `||` operator:**
```bash
#!/bin/bash
set -e

false || true  # Script continues!
echo "still running"
```

**3. In test conditions:**
```bash
#!/bin/bash
set -e

# This doesn't exit
[ -f /nonexistent/file ]

echo "still running"
```

**4. Left side of pipes (without pipefail):**
```bash
#!/bin/bash
set -e

false | true  # Script continues!
echo "still running"
```

### When `set -e` DOES Exit

**1. Simple command failure:**
```bash
#!/bin/bash
set -e

false  # Script exits here
echo "won't run"
```

**2. In `&&` chain:**
```bash
#!/bin/bash
set -e

true && false  # Script exits here
echo "won't run"
```

**3. Command substitution:**
```bash
#!/bin/bash
set -e

result=$(false)  # Script exits here
echo "won't run"
```

### Proper Error Handling Pattern

**Golden pattern:**

```bash
#!/bin/bash
set -euo pipefail

# Explicit error handling
command || {
    echo "ERROR: Command failed"
    exit 1
}

# With cleanup
command || {
    echo "ERROR: Command failed"
    cleanup
    exit 1
}
```

**Function error handling:**
```bash
#!/bin/bash
set -euo pipefail

deploy() {
    echo "Deploying application..."
    
    # Critical steps with explicit error handling
    docker pull myapp:latest || {
        echo "ERROR: Failed to pull image"
        return 1
    }
    
    docker stop myapp || true  # OK if not running
    
    docker run -d --name myapp myapp:latest || {
        echo "ERROR: Failed to start container"
        return 1
    }
    
    return 0
}

# Use the function
if deploy; then
    echo "Deployment successful"
else
    echo "Deployment failed"
    exit 1
fi
```

### Temporarily Disable Error Exit

```bash
#!/bin/bash
set -e

# Disable temporarily
set +e

# Command that may fail
optional_command
status=$?

# Re-enable
set -e

# Handle the result
if [ $status -ne 0 ]; then
    echo "Optional command failed, continuing..."
fi
```

---

## Signals and Traps (Real Cases)

### Common Signals Reference

| Signal | Number | Meaning | Can Trap? |
|--------|--------|---------|-----------|
| INT | 2 | Ctrl+C | ✅ Yes |
| TERM | 15 | Termination | ✅ Yes |
| KILL | 9 | Force kill | ❌ No |
| HUP | 1 | Hangup | ✅ Yes |
| QUIT | 3 | Quit | ✅ Yes |
| EXIT | - | Script exit | ✅ Yes |
| ERR | - | Command error | ✅ Yes |

### Cleanup on EXIT (Must Use)

**Basic cleanup:**
```bash
#!/bin/bash

cleanup() {
    rm -f /tmp/file.$$
    rm -rf /tmp/workdir.$$
}

trap cleanup EXIT

# Create temp resources
touch /tmp/file.$$
mkdir /tmp/workdir.$$

# Do work...

# Cleanup runs on:
# - Normal exit
# - Error exit
# - Ctrl+C
```

**Production cleanup:**
```bash
#!/bin/bash
set -euo pipefail

# Temp file for storing PIDs
LOCKFILE="/var/run/myapp.lock"
TMPDIR="/tmp/myapp.$$"

cleanup() {
    local exit_code=$?
    
    echo "Cleaning up (exit code: $exit_code)..."
    
    # Kill background processes
    if [ -f "$LOCKFILE" ]; then
        while read pid; do
            kill "$pid" 2>/dev/null || true
        done < "$LOCKFILE"
        rm -f "$LOCKFILE"
    fi
    
    # Remove temp directory
    rm -rf "$TMPDIR"
    
    # Log completion
    echo "Cleanup complete"
    
    exit $exit_code
}

trap cleanup EXIT

# Setup
mkdir -p "$TMPDIR"

# Start background processes
some_daemon &
echo $! >> "$LOCKFILE"

# Do work...
```

### Debug Error Location (Interview Gold)

```bash
#!/bin/bash
set -e

trap 'echo "ERROR at line $LINENO"' ERR

# Script code...
false  # This will trigger: ERROR at line 5

# More code...
```

**Better error reporting:**
```bash
#!/bin/bash
set -e

error_handler() {
    echo "================================================"
    echo "ERROR occurred in script $0"
    echo "Line: $1"
    echo "Exit code: $2"
    echo "Command: $3"
    echo "================================================"
}

trap 'error_handler $LINENO $? "$BASH_COMMAND"' ERR

# Script code...
```

### Handle Ctrl+C Gracefully

```bash
#!/bin/bash

interrupted=false

handle_interrupt() {
    echo ""
    echo "Caught interrupt signal"
    interrupted=true
}

trap handle_interrupt INT

echo "Processing items (Ctrl+C to stop)..."

for i in {1..100}; do
    if $interrupted; then
        echo "Stopping gracefully at item $i"
        break
    fi
    
    echo "Processing item $i"
    sleep 1
done

echo "Done"
```

### Advanced: Reload on SIGHUP

**Daemon-like behavior:**

```bash
#!/bin/bash

CONFIG_FILE="/etc/myapp/config.conf"

load_config() {
    echo "Loading configuration from $CONFIG_FILE"
    source "$CONFIG_FILE"
    echo "Configuration loaded"
}

reload_handler() {
    echo "Received SIGHUP - reloading configuration"
    load_config
}

trap reload_handler HUP

# Initial load
load_config

# Main loop
while true; do
    echo "Working with config: $MY_SETTING"
    sleep 10
done

# Send reload signal:
# kill -HUP <pid>
```

---

## Defensive Scripting Patterns

### Always Quote Variables

**❌ Dangerous:**
```bash
#!/bin/bash

file="my file.txt"
rm $file  # Tries to delete "my" and "file.txt" separately!
```

**✅ Safe:**
```bash
#!/bin/bash

file="my file.txt"
rm "$file"  # Correctly deletes "my file.txt"

# Even safer with --
rm -- "$file"  # Protects against files starting with -
```

**Why `--` matters:**
```bash
#!/bin/bash

# File named "-rf"
file="-rf"

rm $file      # DISASTER! Becomes: rm -rf
rm -- "$file" # Safe: deletes file named "-rf"
```

### Validate All Inputs

**Check argument count:**
```bash
#!/bin/bash

if [ $# -ne 2 ]; then
    echo "Usage: $0 <source> <destination>"
    exit 1
fi

source=$1
dest=$2

# Now safe to use
```

**Validate file exists:**
```bash
#!/bin/bash

input_file=$1

if [ ! -f "$input_file" ]; then
    echo "ERROR: File not found: $input_file"
    exit 1
fi

# Process file
```

**Validate directory:**
```bash
#!/bin/bash

backup_dir=$1

if [ ! -d "$backup_dir" ]; then
    echo "ERROR: Directory not found: $backup_dir"
    exit 1
fi

if [ ! -w "$backup_dir" ]; then
    echo "ERROR: Directory not writable: $backup_dir"
    exit 1
fi

# Proceed with backup
```

### Use Absolute Paths (Cron-Safe)

**Cron has limited `$PATH`:**

```bash
#!/bin/bash

# ❌ Bad (may fail in cron)
cp file.txt backup/
rm file.txt

# ✅ Good (works everywhere)
/bin/cp /data/file.txt /backup/
/bin/rm /data/file.txt
```

**Define paths at top:**
```bash
#!/bin/bash

# Absolute paths
readonly CP=/bin/cp
readonly RM=/bin/rm
readonly TAR=/bin/tar
readonly MYSQL=/usr/bin/mysql

# Use throughout script
$CP file.txt backup/
$TAR -czf backup.tar.gz data/
```

### Use `readonly` for Constants

**Prevent accidental overwrites:**

```bash
#!/bin/bash

readonly CONFIG_FILE="/etc/myapp/config.conf"
readonly LOG_FILE="/var/log/myapp.log"
readonly DATA_DIR="/var/lib/myapp"

# These cannot be changed
CONFIG_FILE="/tmp/test"  # ERROR: readonly variable
```

**Function-local constants:**
```bash
#!/bin/bash

deploy() {
    local readonly IMAGE="myapp:latest"
    local readonly CONTAINER="myapp"
    
    docker pull "$IMAGE"
    docker run -d --name "$CONTAINER" "$IMAGE"
}
```

### Parameter Expansion Defaults

```bash
#!/bin/bash

# Use default if variable is unset
environment=${1:-production}

# Use default if variable is unset or empty
port=${PORT:-8080}

# Error if variable is unset
database=${DATABASE:?ERROR: DATABASE not set}

# Set and use default
log_level=${LOG_LEVEL:=INFO}
```

---

## Idempotency

**Critical DevOps Concept:**
> **Running the script multiple times should give the same result**

### Non-Idempotent vs Idempotent

**❌ Non-idempotent:**
```bash
#!/bin/bash

# Fails on second run
useradd appuser
mkdir /opt/app
systemctl start myapp
```

**✅ Idempotent:**
```bash
#!/bin/bash

# Safe to run multiple times

# Create user if doesn't exist
if ! id appuser &>/dev/null; then
    useradd -m -s /bin/bash appuser
fi

# Create directory if doesn't exist
mkdir -p /opt/app

# Start service if not running
if ! systemctl is-active --quiet myapp; then
    systemctl start myapp
fi
```

### Idempotent Patterns

**1. Directory creation:**
```bash
# Non-idempotent
mkdir /opt/app  # Fails if exists

# Idempotent
mkdir -p /opt/app
```

**2. User creation:**
```bash
# Non-idempotent
useradd myuser  # Fails if exists

# Idempotent
id myuser &>/dev/null || useradd -m myuser
```

**3. Package installation:**
```bash
# Non-idempotent
apt install nginx  # May fail if already installed

# Idempotent (Debian/Ubuntu)
dpkg -l nginx &>/dev/null || apt install -y nginx

# Idempotent (RHEL/CentOS)
rpm -q nginx &>/dev/null || yum install -y nginx
```

**4. File configuration:**
```bash
#!/bin/bash

CONFIG_FILE="/etc/myapp/config.conf"
SETTING="server=localhost"

# Idempotent: Update or add setting
if grep -q "^server=" "$CONFIG_FILE"; then
    # Update existing
    sed -i "s/^server=.*/server=localhost/" "$CONFIG_FILE"
else
    # Add new
    echo "$SETTING" >> "$CONFIG_FILE"
fi
```

**5. Symlink creation:**
```bash
# Non-idempotent
ln -s /opt/app/current /var/www/app  # Fails if exists

# Idempotent
ln -sfn /opt/app/current /var/www/app
# -s: symbolic link
# -f: force (remove existing)
# -n: treat link as normal file if it's a symlink to directory
```

**6. Systemd service:**
```bash
#!/bin/bash

SERVICE="myapp"

# Idempotent enable
systemctl is-enabled --quiet "$SERVICE" || systemctl enable "$SERVICE"

# Idempotent start
systemctl is-active --quiet "$SERVICE" || systemctl start "$SERVICE"
```

### Complete Idempotent Setup Script

```bash
#!/bin/bash
set -euo pipefail

APP_USER="myapp"
APP_DIR="/opt/myapp"
CONFIG_FILE="/etc/myapp/config.conf"

echo "Setting up application..."

# Create user
if ! id "$APP_USER" &>/dev/null; then
    echo "Creating user $APP_USER"
    useradd -r -s /sbin/nologin "$APP_USER"
fi

# Create directories
mkdir -p "$APP_DIR"/{bin,config,data,logs}
mkdir -p /etc/myapp

# Set ownership
chown -R "$APP_USER:$APP_USER" "$APP_DIR"

# Install package if needed
if ! command -v nginx &>/dev/null; then
    echo "Installing nginx"
    apt update && apt install -y nginx
fi

# Configure
if [ ! -f "$CONFIG_FILE" ]; then
    echo "Creating config file"
    cat > "$CONFIG_FILE" << EOF
server=localhost
port=8080
user=$APP_USER
EOF
fi

# Enable and start service
systemctl is-enabled --quiet nginx || systemctl enable nginx
systemctl is-active --quiet nginx || systemctl start nginx

echo "Setup complete (idempotent)"
```

---

## Debugging Techniques

### Debug Mode (`set -x`)

**Enable tracing:**
```bash
#!/bin/bash

set -x  # Enable debug mode

echo "Starting script"
ls /tmp
date

# Output shows every command:
# + echo 'Starting script'
# Starting script
# + ls /tmp
# file1 file2
# + date
# Tue Jan 15 10:30:00 UTC 2024
```

### Trace Specific Section

```bash
#!/bin/bash

echo "Normal output"

set -x  # Enable tracing
complex_operation
critical_section
set +x  # Disable tracing

echo "Back to normal"
```

### Debug Function

```bash
#!/bin/bash

DEBUG=${DEBUG:-0}

debug() {
    if [ "$DEBUG" = "1" ]; then
        echo "DEBUG: $*" >&2
    fi
}

# Usage
debug "Starting backup process"
debug "Source: $SOURCE_DIR"
debug "Destination: $BACKUP_DIR"

# Run with: DEBUG=1 ./script.sh
```

### Dry Run Pattern

**Test without making changes:**

```bash
#!/bin/bash

DRY_RUN=${DRY_RUN:-false}

run() {
    if [ "$DRY_RUN" = "true" ]; then
        echo "DRY RUN: $*"
    else
        "$@"
    fi
}

# Usage
run rm -rf /data
run systemctl restart nginx
run mv file.txt backup/

# Execute:
# DRY_RUN=true ./script.sh  # Just prints commands
# ./script.sh               # Actually runs commands
```

**Production dry-run:**
```bash
#!/bin/bash

DRY_RUN=${DRY_RUN:-false}

execute() {
    echo "Executing: $*"
    if [ "$DRY_RUN" = "true" ]; then
        echo "  (dry run - not executed)"
        return 0
    fi
    "$@"
}

# Deployment script
execute docker pull myapp:latest
execute docker stop myapp
execute docker rm myapp
execute docker run -d --name myapp myapp:latest

# Test: DRY_RUN=true ./deploy.sh
# Real: ./deploy.sh
```

### Verbose Mode

```bash
#!/bin/bash

VERBOSE=${VERBOSE:-0}

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

verbose() {
    if [ "$VERBOSE" = "1" ]; then
        log "VERBOSE: $*"
    fi
}

# Usage
log "Starting backup"
verbose "Source directory: $SOURCE"
verbose "Backup directory: $DEST"
verbose "Retention days: $RETENTION"

# Run: VERBOSE=1 ./script.sh
```

---

## Production-Ready Script Example

**Full-featured production script:**

```bash
#!/bin/bash
#
# Backup Script - Production Quality
# Author: DevOps Team
# Description: Creates incremental backups with logging and monitoring
#

set -euo pipefail

#==============================================
# Configuration
#==============================================
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_DIR=$(dirname "$(readlink -f "$0")")
readonly LOG_FILE="/var/log/backup.log"
readonly LOCK_FILE="/var/run/backup.lock"

readonly SOURCE_DIR="/data"
readonly BACKUP_DIR="/backups"
readonly RETENTION_DAYS=7

#==============================================
# Functions
#==============================================

log() {
    local level=$1
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a "$LOG_FILE"
}

cleanup() {
    local exit_code=$?
    
    log "INFO" "Cleanup started"
    
    # Remove lock file
    rm -f "$LOCK_FILE"
    
    if [ $exit_code -eq 0 ]; then
        log "INFO" "Script completed successfully"
    else
        log "ERROR" "Script failed with exit code $exit_code"
    fi
    
    exit $exit_code
}

check_lock() {
    if [ -f "$LOCK_FILE" ]; then
        log "ERROR" "Another instance is running (lock file exists)"
        exit 1
    fi
    echo $$ > "$LOCK_FILE"
}

validate_environment() {
    log "INFO" "Validating environment"
    
    # Check if running as root
    if [ $EUID -ne 0 ]; then
        log "ERROR" "This script must be run as root"
        exit 1
    fi
    
    # Check source directory
    if [ ! -d "$SOURCE_DIR" ]; then
        log "ERROR" "Source directory not found: $SOURCE_DIR"
        exit 1
    fi
    
    # Create backup directory if needed
    mkdir -p "$BACKUP_DIR"
    
    # Check disk space (need at least 10GB free)
    local free_space=$(df "$BACKUP_DIR" | awk 'NR==2 {print $4}')
    if [ "$free_space" -lt 10485760 ]; then  # 10GB in KB
        log "ERROR" "Insufficient disk space in $BACKUP_DIR"
        exit 1
    fi
    
    log "INFO" "Environment validation passed"
}

perform_backup() {
    log "INFO" "Starting backup"
    
    local backup_name="backup-$(date +%Y%m%d-%H%M%S)"
    local backup_path="$BACKUP_DIR/$backup_name"
    
    # Create backup
    if tar -czf "$backup_path.tar.gz" "$SOURCE_DIR" 2>> "$LOG_FILE"; then
        log "INFO" "Backup created: $backup_name.tar.gz"
        
        # Get backup size
        local size=$(du -h "$backup_path.tar.gz" | cut -f1)
        log "INFO" "Backup size: $size"
    else
        log "ERROR" "Backup creation failed"
        return 1
    fi
}

cleanup_old_backups() {
    log "INFO" "Cleaning up old backups (retention: $RETENTION_DAYS days)"
    
    local count=$(find "$BACKUP_DIR" -name "backup-*.tar.gz" -mtime +$RETENTION_DAYS | wc -l)
    
    if [ "$count" -gt 0 ]; then
        find "$BACKUP_DIR" -name "backup-*.tar.gz" -mtime +$RETENTION_DAYS -delete
        log "INFO" "Removed $count old backup(s)"
    else
        log "INFO" "No old backups to remove"
    fi
}

send_notification() {
    local status=$1
    local message=$2
    
    # Send email notification (if msmtp configured)
    if command -v msmtp &>/dev/null; then
        {
            echo "Subject: Backup $status - $(hostname)"
            echo ""
            echo "$message"
            echo ""
            echo "Timestamp: $(date)"
            echo "Hostname: $(hostname)"
        } | msmtp admin@example.com
        
        log "INFO" "Notification sent"
    fi
}

#==============================================
# Main
#==============================================

main() {
    log "INFO" "=== Backup script started ==="
    
    # Setup signal handlers
    trap cleanup EXIT
    trap 'log "ERROR" "Script interrupted"; exit 130' INT TERM
    
    # Pre-flight checks
    check_lock
    validate_environment
    
    # Perform backup
    if perform_backup; then
        cleanup_old_backups
        send_notification "SUCCESS" "Backup completed successfully"
    else
        send_notification "FAILED" "Backup failed - check logs"
        exit 1
    fi
    
    log "INFO" "=== Backup script completed ==="
}

# Execute main function
main "$@"
```

---

## Interview One-Liners

**Memorize these for interviews:**

### 1. Error Handling
```
set -euo pipefail prevents silent failures
```

### 2. Idempotency
```
Idempotency is critical in automation - scripts should be safe to run multiple times
```

### 3. Trap
```
Always trap EXIT for cleanup - runs on success, failure, and interruption
```

### 4. Input Validation
```
Never trust user input - always validate and quote variables
```

### 5. Quoting
```
Always quote variables: "$var" not $var
```

### 6. Atomic Operations
```
Use atomic file writes (temp + mv) to prevent corruption
```

### 7. Debugging
```
Use 'set -x' for debugging and dry-run patterns for testing
```

### 8. Absolute Paths
```
Use absolute paths in cron jobs - cron has limited $PATH
```

### 9. Lock Files
```
Use lock files to prevent concurrent execution
```

### 10. Logging
```
Always log to file with timestamps for troubleshooting
```

---

## Quick Reference: Production Checklist

**Before deploying a script to production:**

- [ ] `set -euo pipefail` at the top
- [ ] `trap cleanup EXIT` for cleanup
- [ ] All variables quoted: `"$var"`
- [ ] Input validation
- [ ] Lock file to prevent concurrent runs
- [ ] Logging with timestamps
- [ ] Error handling with meaningful messages
- [ ] Idempotent operations
- [ ] Dry-run mode for testing
- [ ] Comments explaining complex logic
- [ ] Absolute paths for cron compatibility

---

This advanced guide covers production-grade shell scripting techniques used in professional DevOps and infra automation!