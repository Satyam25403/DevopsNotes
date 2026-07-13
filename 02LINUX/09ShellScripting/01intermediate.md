# Intermediate Shell Scripting

Advanced shell scripting techniques including error handling, safety features, practical examples, and email integration.

## Table of Contents
- [Error Handling and Safety](#error-handling-and-safety)
- [Trap and Signal Handling](#trap-and-signal-handling)
- [Practical Production Scripts](#practical-production-scripts)
- [Email Integration with msmtp](#email-integration-with-msmtp)
- [Array Handling](#array-handling)
- [String Manipulation](#string-manipulation)
- [Advanced File Operations](#advanced-file-operations)
- [Process Management](#process-management)

---

## Error Handling and Safety

### set -e: Exit on Error

**Exit immediately if any command fails:**

```bash
#!/bin/bash
set -e

# Script stops if any command fails
mkdir /some/dir
cd /some/dir
cp file.txt backup.txt

# If mkdir fails, script stops immediately
```

**When `set -e` does NOT work:**
```bash
set -e

# Does NOT exit in conditional
if false; then
    echo "won't run"
fi
echo "still runs"  # This executes!

# Does NOT exit with ||
false || true  # This continues

# Does NOT exit in pipelines (without pipefail)
false | true  # Exits with 0
```

### set -u: Undefined Variable Protection

**Treats unset variables as error:**

```bash
#!/bin/bash
set -u

name="Alice"
echo "$name"      # OK

echo "$age"       # ERROR: age: unbound variable
```

**Use case:**
```bash
#!/bin/bash
set -u

# This catches typos
usrname="admin"   # Typo!
echo "$username"  # ERROR - prevents silent bugs
```

### set -o pipefail: Catch Pipe Failures

**Without pipefail:**
```bash
#!/bin/bash
set -e

false | true
echo $?  # Output: 0 (only last command checked)
```

**With pipefail:**
```bash
#!/bin/bash
set -e
set -o pipefail

false | true
echo $?  # Non-zero (first failure caught)
```

**Real-world example:**
```bash
#!/bin/bash
set -eo pipefail

# If grep fails to find pattern, script stops
cat logfile.txt | grep "ERROR" | wc -l
```

### The Golden Combo

**Use all three together:**

```bash
#!/bin/bash
set -euo pipefail

# Now your script is:
# - Exit on error (-e)
# - Catch undefined variables (-u)
# - Catch pipe failures (-o pipefail)
```

### Conditional Error Handling

**Handle expected failures:**

```bash
#!/bin/bash
set -e

# Temporarily disable exit on error
set +e
command_that_might_fail
status=$?
set -e

if [ $status -ne 0 ]; then
    echo "Command failed with status $status"
    # Handle error gracefully
fi
```

**Or use explicit handling:**
```bash
#!/bin/bash
set -e

# This won't exit script
command || {
    echo "Command failed"
    exit 1
}

# Continue if command succeeded
echo "Success"
```

---

## Trap and Signal Handling

### Understanding Signals

**Common signals:**

| Signal | Number | Meaning | Source |
|--------|--------|---------|--------|
| INT | 2 | Interrupt | Ctrl+C |
| TERM | 15 | Terminate | `kill` |
| KILL | 9 | Force kill | `kill -9` (cannot trap) |
| EXIT | - | Script exit | Normal/abnormal exit |
| ERR | - | Command error | Failed command |

### Basic Trap Usage

```bash
#!/bin/bash

# Trap on script exit
trap 'echo "Script exited"' EXIT

echo "Running..."
# Script exits, trap executes
```

### Cleanup on Exit

**Always cleanup temporary files:**

```bash
#!/bin/bash

# Create temp file with unique name
tmpfile="/tmp/data.$$"  # $$ = current PID

# Setup cleanup trap
trap 'rm -f "$tmpfile"' EXIT

# Use temp file
echo "data" > "$tmpfile"
# Process file...

# Cleanup happens automatically on exit (success or failure)
```

**More comprehensive cleanup:**
```bash
#!/bin/bash

cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/lockfile.$$
    rm -rf /tmp/workdir.$$
    echo "Cleanup complete"
}

trap cleanup EXIT

# Create temporary resources
touch /tmp/lockfile.$$
mkdir /tmp/workdir.$$

# Do work...

# Cleanup happens automatically
```

### Handle Ctrl+C (SIGINT)

```bash
#!/bin/bash

handle_interrupt() {
    echo ""
    echo "Script interrupted by user"
    exit 1
}

trap handle_interrupt INT

echo "Running long process..."
echo "Press Ctrl+C to interrupt"

# Simulate long-running process
for i in {1..100}; do
    echo "Processing $i/100..."
    sleep 1
done

echo "Process complete"
```

### Trap on Error

**Catch and report errors:**

```bash
#!/bin/bash
set -e

trap 'echo "Error occurred at line $LINENO"' ERR

echo "Starting script..."
# Some commands...
false  # This triggers ERR trap
echo "This won't execute"
```

**Better error reporting:**
```bash
#!/bin/bash
set -e

error_handler() {
    echo "ERROR: Command failed at line $1"
    echo "Exit code: $2"
    exit $2
}

trap 'error_handler $LINENO $?' ERR

# Your script...
```

### Multiple Signal Handling

```bash
#!/bin/bash

cleanup() {
    echo "Cleaning up..."
    # Cleanup code
}

handle_interrupt() {
    echo "Interrupted!"
    cleanup
    exit 130
}

# Trap multiple signals
trap cleanup EXIT
trap handle_interrupt INT TERM

# Script logic...
```

---

## Practical Production Scripts

### 1. Service Check and Restart

```bash
#!/bin/bash
set -e

SERVICE="nginx"

if systemctl is-active "$SERVICE" >/dev/null 2>&1; then
    echo "$SERVICE is running"
else
    echo "$SERVICE is down, restarting..."
    systemctl restart "$SERVICE"
    
    # Verify restart
    sleep 2
    if systemctl is-active "$SERVICE" >/dev/null 2>&1; then
        echo "$SERVICE restarted successfully"
    else
        echo "ERROR: Failed to restart $SERVICE"
        exit 1
    fi
fi
```

### 2. File Backup with Timestamp

```bash
#!/bin/bash
set -e

SRC="/etc/nginx/nginx.conf"
DEST="/backup"

# Validate source exists
if [ ! -f "$SRC" ]; then
    echo "ERROR: Source file not found: $SRC"
    exit 1
fi

# Create backup directory if needed
mkdir -p "$DEST"

# Create timestamped backup
TIMESTAMP=$(date +%F_%T)
BACKUP_FILE="$DEST/nginx.conf.$TIMESTAMP"

cp "$SRC" "$BACKUP_FILE"
echo "Backup created: $BACKUP_FILE"
```

### 3. Disk Usage Alert

```bash
#!/bin/bash
set -e

THRESHOLD=80

# Get disk usage percentage
USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')

if [ "$USAGE" -gt "$THRESHOLD" ]; then
    echo "ALERT: Disk usage critical: ${USAGE}%"
    echo "Threshold: ${THRESHOLD}%"
    
    # Show largest directories
    echo "Largest directories:"
    du -sh /* 2>/dev/null | sort -rh | head -5
    
    exit 1
else
    echo "Disk usage normal: ${USAGE}%"
fi
```

### 4. Process Log Files

```bash
#!/bin/bash
set -e

# Process all log files
for file in *.log; do
    # Skip if no log files exist
    [ -f "$file" ] || continue
    
    echo "Processing $file..."
    
    # Count errors
    errors=$(grep -c "ERROR" "$file" 2>/dev/null || echo 0)
    
    # Count warnings
    warnings=$(grep -c "WARN" "$file" 2>/dev/null || echo 0)
    
    echo "  Errors: $errors"
    echo "  Warnings: $warnings"
    
    # Archive if old
    if [ -f "$file" ] && [ $(find "$file" -mtime +7) ]; then
        gzip "$file"
        echo "  Archived: ${file}.gz"
    fi
    
    echo ""
done
```

### 5. UFW Firewall Rule Deletion

```bash
#!/bin/bash
set -euo pipefail

# Enable firewall and show rules
sudo ufw enable
sudo ufw status numbered

# Get user input
read -p "Which UFW rules do you want to delete? (e.g., 1 2 4 6): " choices

# Validate input (numbers only)
for choice in $choices; do
    if [[ ! "$choice" =~ ^[0-9]+$ ]]; then
        echo "ERROR: Invalid input: $choice (not a number)"
        exit 1
    fi
done

# Sort in reverse order (delete from bottom to avoid renumbering)
sorted_choices=$(echo "$choices" | tr ' ' '\n' | sort -nr)

# Delete rules
for rule in $sorted_choices; do
    echo "Deleting rule $rule..."
    sudo ufw delete "$rule"
done

echo "Selected UFW rules deleted successfully"

# Show updated rules
sudo ufw status numbered
```

### 6. Backup Script with Logging

```bash
#!/usr/bin/env bash
set -euo pipefail

# Get user input
read -p "Source directory to backup: " SRC_DIR
read -p "Backup destination: " BACKUP_DIR
read -p "Log file location: " LOG_FILE

# Validate source
if [[ ! -d "$SRC_DIR" ]]; then
    echo "ERROR: Source directory does not exist: $SRC_DIR"
    exit 1
fi

# Create backup directory
DATE=$(date +%Y-%m-%d)
FULL_BACKUP_DIR="${BACKUP_DIR}/backup-${DATE}"
mkdir -p "$FULL_BACKUP_DIR"

# Create log directory
mkdir -p "$(dirname "$LOG_FILE")"

# Log function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >> "$LOG_FILE"
}

log "Backup started"
log "Source: $SRC_DIR"
log "Destination: $FULL_BACKUP_DIR"

# Perform backup with rsync
if rsync -av --delete "$SRC_DIR/" "$FULL_BACKUP_DIR/" >> "$LOG_FILE" 2>&1; then
    log "Backup completed successfully"
    echo "✅ Backup completed successfully"
else
    log "ERROR: Backup failed"
    echo "❌ Backup failed. Check log: $LOG_FILE"
    exit 1
fi
```

### 7. Service Monitoring Script

```bash
#!/usr/bin/env bash
set -euo pipefail

# Check root
if [[ $EUID -ne 0 ]]; then
    echo "ERROR: This script must be run as root"
    exit 1
fi

# Get services to monitor
read -p "Enter services to monitor (comma-separated): " SERVICE_INPUT

# Split input into array
IFS=',' read -ra SERVICES <<< "$SERVICE_INPUT"

# Monitor each service
for service in "${SERVICES[@]}"; do
    # Trim whitespace
    service=$(echo "$service" | xargs)
    
    echo "Checking $service..."
    
    if systemctl is-active --quiet "$service"; then
        echo "  ✅ $service is running"
    else
        echo "  ⚠️  $service is not running - attempting restart..."
        
        if systemctl start "$service"; then
            echo "  ✅ $service restarted successfully"
        else
            echo "  ❌ Failed to restart $service"
        fi
    fi
done

echo ""
echo "Monitoring complete"
```

---

## Email Integration with msmtp

### Installing msmtp

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install msmtp msmtp-mta

# Verify installation
which msmtp
```

### Configuring msmtp

**Create config file: `~/.msmtprc`**

```bash
# General defaults
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        ~/.msmtp.log

# Gmail account
account gmail
host smtp.gmail.com
port 587
from yourmail@gmail.com
user yourmail@gmail.com
password YOUR_APP_PASSWORD

# Set default account
account default : gmail
```

**Secure the config file:**
```bash
chmod 600 ~/.msmtprc
```

**For Gmail:** Generate App Password:
1. Go to Google Account settings
2. Security → 2-Step Verification
3. App Passwords
4. Generate password for "Mail"
5. Use generated password in config

### Common SMTP Providers

| Provider | SMTP Host | Port | TLS |
|----------|-----------|------|-----|
| Gmail | smtp.gmail.com | 587 | Yes |
| Outlook | smtp.office365.com | 587 | Yes |
| Yahoo | smtp.mail.yahoo.com | 587 | Yes |
| AWS SES | email-smtp.{region}.amazonaws.com | 587 | Yes |
| SendGrid | smtp.sendgrid.net | 587 | Yes |

### Test Email

```bash
# Simple test
echo -e "Subject: Test Mail\n\nHello from Linux" | msmtp recipient@example.com

# With body
echo -e "Subject: Test\n\nThis is a test message" | msmtp -a gmail recipient@example.com
```

### Send Email Script

**Basic email script:**

```bash
#!/usr/bin/env bash

TO="recipient@example.com"
FROM="sender@example.com"
SUBJECT="Server Alert"
BODY="This is an automated alert from $(hostname)"

# Send email
{
    echo "From: $FROM"
    echo "To: $TO"
    echo "Subject: $SUBJECT"
    echo ""
    echo "$BODY"
} | msmtp "$TO"

echo "Email sent to $TO"
```

### Disk Alert with Email

```bash
#!/bin/bash
set -e

TO="admin@example.com"
THRESHOLD=80

# Check disk usage
USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')

if [ "$USAGE" -gt "$THRESHOLD" ]; then
    SUBJECT="ALERT: High Disk Usage on $(hostname)"
    BODY="Disk usage is ${USAGE}% (threshold: ${THRESHOLD}%)\n\n"
    BODY+="Top disk consumers:\n"
    BODY+="$(du -sh /* 2>/dev/null | sort -rh | head -5)"
    
    # Send alert email
    {
        echo "Subject: $SUBJECT"
        echo ""
        echo -e "$BODY"
    } | msmtp "$TO"
    
    echo "Alert email sent"
fi
```

### Backup Notification Script

```bash
#!/bin/bash
set -e

TO="admin@example.com"
BACKUP_DIR="/backup"
LOG_FILE="/var/log/backup.log"

# Perform backup
if tar -czf "$BACKUP_DIR/backup-$(date +%F).tar.gz" /data 2>> "$LOG_FILE"; then
    STATUS="SUCCESS"
    SUBJECT="Backup Successful - $(hostname)"
    BODY="Backup completed successfully at $(date)"
else
    STATUS="FAILED"
    SUBJECT="Backup Failed - $(hostname)"
    BODY="Backup failed at $(date)\n\nCheck logs: $LOG_FILE"
fi

# Send notification
{
    echo "Subject: $SUBJECT"
    echo ""
    echo -e "$BODY"
} | msmtp "$TO"
```

---

## Array Handling

### Creating Arrays

```bash
#!/bin/bash

# Method 1: Declare array
fruits=("apple" "banana" "orange")

# Method 2: Individual assignment
colors[0]="red"
colors[1]="blue"
colors[2]="green"

# Method 3: From command output
files=($(ls *.txt))

# Method 4: Read into array
IFS=',' read -ra items <<< "item1,item2,item3"
```

### Accessing Array Elements

```bash
#!/bin/bash

fruits=("apple" "banana" "orange")

# Access element
echo "${fruits[0]}"      # apple
echo "${fruits[1]}"      # banana

# All elements
echo "${fruits[@]}"      # apple banana orange
echo "${fruits[*]}"      # apple banana orange

# Array length
echo "${#fruits[@]}"     # 3

# Indices
echo "${!fruits[@]}"     # 0 1 2
```

### Iterating Over Arrays

```bash
#!/bin/bash

services=("nginx" "mysql" "redis")

# Method 1: For loop
for service in "${services[@]}"; do
    echo "Checking $service"
    systemctl status "$service"
done

# Method 2: Index-based
for i in "${!services[@]}"; do
    echo "Service $i: ${services[$i]}"
done
```

### Array from CSV

```bash
#!/bin/bash

# Read comma-separated input
read -p "Enter services (comma-separated): " input

# Split into array
IFS=',' read -ra services <<< "$input"

# Process each service
for service in "${services[@]}"; do
    # Trim whitespace
    service=$(echo "$service" | xargs)
    echo "Processing: $service"
done
```

### Adding to Arrays

```bash
#!/bin/bash

fruits=("apple")

# Add element
fruits+=("banana")
fruits+=("orange")

echo "${fruits[@]}"  # apple banana orange
```

---

## String Manipulation

### String Length

```bash
#!/bin/bash

text="Hello World"
echo "${#text}"  # 11
```

### Substring Extraction

```bash
#!/bin/bash

text="Hello World"

# Extract substring
echo "${text:0:5}"   # Hello
echo "${text:6}"     # World
echo "${text:6:5}"   # World
```

### String Replacement

```bash
#!/bin/bash

text="Hello World World"

# Replace first occurrence
echo "${text/World/Universe}"  # Hello Universe World

# Replace all occurrences
echo "${text//World/Universe}"  # Hello Universe Universe
```

### String Case Conversion

```bash
#!/bin/bash

text="Hello World"

# Uppercase
echo "${text^^}"  # HELLO WORLD

# Lowercase
echo "${text,,}"  # hello world

# First letter uppercase
echo "${text^}"   # Hello world
```

### String Trimming

```bash
#!/bin/bash

# Remove leading/trailing whitespace
text="  hello world  "
trimmed=$(echo "$text" | xargs)
echo "$trimmed"  # hello world
```

---

## Advanced File Operations

### Find and Process Files

```bash
#!/bin/bash

# Find files modified in last 7 days
find /var/log -name "*.log" -mtime -7 -exec ls -lh {} \;

# Find and delete old files
find /tmp -name "*.tmp" -mtime +30 -delete

# Find large files
find /home -type f -size +100M -exec ls -lh {} \;
```

### Reading Files Line by Line

```bash
#!/bin/bash

while IFS= read -r line; do
    echo "Processing: $line"
done < input.txt
```

### Parallel Processing

```bash
#!/bin/bash

# Process multiple files in parallel
for file in *.log; do
    (
        echo "Processing $file"
        gzip "$file"
    ) &
done

# Wait for all background jobs
wait

echo "All files processed"
```

---

## Process Management

### Background Jobs

```bash
#!/bin/bash

# Run in background
long_running_command &

# Save PID
PID=$!

echo "Started process $PID"

# Wait for completion
wait $PID

echo "Process completed"
```

### Process Monitoring

```bash
#!/bin/bash

# Check if process is running
if pgrep -x "nginx" > /dev/null; then
    echo "Nginx is running"
else
    echo "Nginx is not running"
fi

# Get process PID
PID=$(pgrep -x "nginx")
echo "Nginx PID: $PID"
```

### Kill Process by Name

```bash
#!/bin/bash

PROCESS="firefox"

# Kill process
pkill "$PROCESS"

# Force kill
pkill -9 "$PROCESS"

# Kill all user processes
pkill -u username
```

---

## Best Practices Summary

**Essential practices:**

1. **Always use `set -euo pipefail`**
   ```bash
   #!/bin/bash
   set -euo pipefail
   ```

2. **Always cleanup with trap**
   ```bash
   trap 'rm -f /tmp/file.$$' EXIT
   ```

3. **Quote all variables**
   ```bash
   "$variable"
   ```

4. **Validate inputs**
   ```bash
   if [ $# -ne 2 ]; then
       echo "Usage: $0 <arg1> <arg2>"
       exit 1
   fi
   ```

5. **Use meaningful variable names**
   ```bash
   backup_directory="/backup"  # Good
   bd="/backup"                # Bad
   ```

6. **Add logging**
   ```bash
   log() {
       echo "[$(date)] $*" >> /var/log/script.log
   }
   ```

7. **Check file existence**
   ```bash
   if [ ! -f "$file" ]; then
       echo "File not found: $file"
       exit 1
   fi
   ```

---

This guide covers intermediate shell scripting techniques essential for production automation and DevOps workflows!