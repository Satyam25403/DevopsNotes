# Shell Scripting Basics

Comprehensive guide to getting started with Bash shell scripting in Linux.

## Table of Contents
- [Introduction](#introduction)
- [Getting Started](#getting-started)
- [Variables](#variables)
- [User Input](#user-input)
- [Command-Line Arguments](#command-line-arguments)
- [Comparison Operators](#comparison-operators)
- [Conditionals](#conditionals)
- [Loops](#loops)
- [Functions](#functions)
- [Output Redirection](#output-redirection)
- [Exit Status](#exit-status)
- [Practical Examples](#practical-examples)

---

## Introduction

### What is a Shell Script?

A **shell script** is a file containing Linux commands executed sequentially by a shell (usually bash).

**Key benefits:**
- ✅ Automate repetitive tasks
- ✅ Batch process commands
- ✅ System administration
- ✅ DevOps automation
- ✅ Scheduled tasks (cron)

### Which Shell to Use?

```bash
# Check available shells
cat /etc/shells

# Find your current shell
echo $SHELL

# Find bash location
which bash
```

**Common shells:**
- `/bin/bash` - Bash (most common)
- `/bin/sh` - POSIX shell
- `/bin/zsh` - Z shell
- `/bin/dash` - Debian Almquist shell (lightweight)

---

## Getting Started

### Creating Your First Script

**Step-by-step process:**

```bash
# 0. Check bash location
which bash
# Output: /bin/bash

# 1. Create script file
vi hello.sh
# OR
nano hello.sh

# 2. Write script (see below)

# 3. Make executable
chmod +x hello.sh

# 4. Run script
./hello.sh
```

### Hello World Script

```bash
#!/bin/bash
# Shebang line - tells system which interpreter to use

echo "Hello, World!"
```

**The Shebang (`#!`):**
- Must be first line
- Specifies interpreter path
- Common shebangs:
  - `#!/bin/bash` - Bash
  - `#!/bin/sh` - POSIX shell
  - `#!/usr/bin/env bash` - Portable (finds bash in PATH)

### Script Execution

```bash
# Method 1: Direct execution (needs execute permission)
chmod +x script.sh
./script.sh

# Method 2: Explicit interpreter
bash script.sh
sh script.sh

# Method 3: Source (runs in current shell)
source script.sh
. script.sh
```

---

## Variables

### Defining Variables

```bash
#!/bin/bash

# Define variable (NO spaces around =)
name="Sagar"
age=25
city="New York"

# Use variable
echo "Hello, $name"
echo "Hello, ${name}"        # Recommended (clear boundary)

# Concatenation
fullname="${name} Kumar"
echo "Full name: $fullname"
```

**Important rules:**
- ❌ No spaces: `name = "John"` (wrong)
- ✅ Correct: `name="John"`
- Use `${var}` to avoid ambiguity
- Variables are strings by default

### Variable Types

```bash
# String
name="Alice"

# Number (treated as string)
age=30

# Command substitution
current_date=$(date)
uptime_info=$(uptime)

# Environment variable
export PATH="/usr/local/bin:$PATH"
```

### Command Substitution

**Execute commands and capture output:**

```bash
#!/bin/bash

# Method 1: $()  (recommended)
current_time=$(date +%H:%M:%S)
echo "Current time: $current_time"

# Method 2: backticks (legacy)
hostname=`hostname`
echo "Hostname: $hostname"

# Examples
echo "Uptime: $(uptime)"
echo "Current user: $(whoami)"
echo "Free disk space: $(df -h / | awk 'NR==2 {print $4}')"
```

### Special Variables

```bash
$0      # Script name
$1-$9   # Positional parameters (arguments)
$#      # Number of arguments
$@      # All arguments as separate words
$*      # All arguments as single word
$?      # Exit status of last command
$$      # Current process ID
$!      # PID of last background process
```

---

## User Input

### Reading Input

```bash
#!/bin/bash

# Basic input
echo "Enter your name:"
read name
echo "Hello, $name"

# With prompt (preferred)
read -p "Enter your age: " age
echo "You are $age years old"

# Multiple inputs
read -p "Enter first and last name: " firstname lastname
echo "Hello, $firstname $lastname"
```

### Reading Passwords (Hidden Input)

```bash
#!/bin/bash

# Silent input (for passwords)
read -s -p "Enter password: " password
echo  # New line after password
echo "Password saved"

# Verify password
read -s -p "Confirm password: " password2
echo

if [ "$password" = "$password2" ]; then
    echo "Passwords match"
else
    echo "Passwords do not match"
fi
```

### Input with Default Value

```bash
#!/bin/bash

read -p "Enter environment [dev]: " env
env=${env:-dev}  # Use 'dev' if empty
echo "Environment: $env"
```

---

## Command-Line Arguments

### Accessing Arguments

```bash
#!/bin/bash

echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "Third argument: $3"
echo "All arguments: $@"
echo "Number of arguments: $#"
```

**Run:**
```bash
chmod +x script.sh
./script.sh dev prod staging

# Output:
# Script name: ./script.sh
# First argument: dev
# Second argument: prod
# Third argument: staging
# All arguments: dev prod staging
# Number of arguments: 3
```

### Argument Validation

```bash
#!/bin/bash

# Check if argument provided
if [ $# -eq 0 ]; then
    echo "Usage: $0 <environment>"
    exit 1
fi

environment=$1
echo "Deploying to $environment"
```

### Processing Multiple Arguments

```bash
#!/bin/bash

echo "Processing $# arguments:"

for arg in "$@"; do
    echo "  - $arg"
done
```

---

## Comparison Operators

### Numeric Comparisons

```bash
# Numeric operators
-eq     # Equal to
-ne     # Not equal to
-lt     # Less than
-le     # Less than or equal to
-gt     # Greater than
-ge     # Greater than or equal to
```

**Examples:**
```bash
#!/bin/bash

age=25

if [ $age -eq 25 ]; then
    echo "Age is 25"
fi

if [ $age -ge 18 ]; then
    echo "Adult"
fi

if [ $age -lt 30 ]; then
    echo "Under 30"
fi
```

### String Comparisons

```bash
# String operators
=       # Equal
==      # Equal (same as =)
!=      # Not equal
-z      # String is empty
-n      # String is not empty
<       # Less than (lexicographic)
>       # Greater than (lexicographic)
```

**Examples:**
```bash
#!/bin/bash

name="Alice"

if [ "$name" = "Alice" ]; then
    echo "Name is Alice"
fi

if [ -z "$name" ]; then
    echo "Name is empty"
else
    echo "Name is not empty"
fi

if [ "$name" != "Bob" ]; then
    echo "Name is not Bob"
fi
```

### File Test Operators

```bash
-f file     # Regular file exists
-d dir      # Directory exists
-e path     # Path exists (file or directory)
-r file     # Readable
-w file     # Writable
-x file     # Executable
-s file     # File not empty (size > 0)
-L file     # Symbolic link
```

**Examples:**
```bash
#!/bin/bash

if [ -f file.txt ]; then
    echo "File exists and is a regular file"
fi

if [ -d /etc ]; then
    echo "Directory exists"
fi

if [ -r file.txt ]; then
    echo "File is readable"
fi

if [ -w file.txt ]; then
    echo "File is writable"
fi

if [ -x script.sh ]; then
    echo "Script is executable"
fi

if [ -s file.txt ]; then
    echo "File is not empty"
fi
```

### Logical Operators

```bash
# Logical AND
cmd1 && cmd2        # Run cmd2 only if cmd1 succeeds

# Logical OR
cmd1 || cmd2        # Run cmd2 only if cmd1 fails

# Negation
! condition

# Combining conditions
[ condition1 ] && [ condition2 ]
[ condition1 ] || [ condition2 ]
```

**Examples:**
```bash
#!/bin/bash

# AND
if [ -f file.txt ] && [ -r file.txt ]; then
    echo "File exists and is readable"
fi

# OR
if [ "$USER" = "root" ] || [ "$USER" = "admin" ]; then
    echo "Privileged user"
fi

# Negation
if [ ! -f file.txt ]; then
    echo "File does not exist"
fi
```

---

## Conditionals

### If-Else Statement

```bash
#!/bin/bash

age=20

if [ $age -ge 18 ]; then
    echo "Adult"
else
    echo "Minor"
fi
```

### If-Elif-Else Statement

```bash
#!/bin/bash

score=85

if [ $score -ge 90 ]; then
    echo "Grade: A"
elif [ $score -ge 80 ]; then
    echo "Grade: B"
elif [ $score -ge 70 ]; then
    echo "Grade: C"
elif [ $score -ge 60 ]; then
    echo "Grade: D"
else
    echo "Grade: F"
fi
```

### Case Statement

**Better than multiple if-elif for matching patterns:**

```bash
#!/bin/bash

action=$1

case $action in
    start)
        echo "Starting service..."
        ;;
    stop)
        echo "Stopping service..."
        ;;
    restart)
        echo "Restarting service..."
        ;;
    status)
        echo "Checking status..."
        ;;
    *)
        echo "Unknown command: $0"
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

**Pattern matching in case:**
```bash
#!/bin/bash

file=$1

case $file in
    *.txt)
        echo "Text file"
        ;;
    *.jpg|*.png|*.gif)
        echo "Image file"
        ;;
    *.sh)
        echo "Shell script"
        ;;
    *)
        echo "Unknown file type"
        ;;
esac
```

---

## Loops

### For Loop

**Iterate over list of items:**

```bash
#!/bin/bash

# Basic for loop
for i in 1 2 3 4 5
do
    echo "Number: $i"
done

# Range syntax
for i in {1..5}
do
    echo "Number: $i"
done

# Step increment
for i in {0..10..2}  # 0, 2, 4, 6, 8, 10
do
    echo $i
done

# C-style for loop
for ((i=1; i<=5; i++))
do
    echo "Count: $i"
done
```

**Iterate over files:**
```bash
#!/bin/bash

# Process all .txt files
for file in *.txt
do
    echo "Processing $file"
done

# Process all log files
for log in /var/log/*.log
do
    if [ -f "$log" ]; then
        echo "Log file: $log"
    fi
done
```

**Iterate over command output:**
```bash
#!/bin/bash

# List all users
for user in $(cat /etc/passwd | cut -d: -f1)
do
    echo "User: $user"
done
```

### While Loop

**Loop while condition is true:**

```bash
#!/bin/bash

count=1
while [ $count -le 5 ]
do
    echo "Count: $count"
    count=$((count + 1))
done
```

**Infinite loop with break:**
```bash
#!/bin/bash

count=1
while true
do
    echo "Iteration: $count"
    count=$((count + 1))
    
    if [ $count -gt 10 ]; then
        break
    fi
done
```

**Reading file line by line:**
```bash
#!/bin/bash

while read line
do
    echo "Line: $line"
done < file.txt
```

### Until Loop

**Loop until condition becomes true:**

```bash
#!/bin/bash

count=1
until [ $count -gt 5 ]
do
    echo "Count: $count"
    count=$((count + 1))
done
```

### Loop Control

```bash
# Break - exit loop
for i in {1..10}
do
    if [ $i -eq 5 ]; then
        break
    fi
    echo $i
done

# Continue - skip to next iteration
for i in {1..10}
do
    if [ $i -eq 5 ]; then
        continue
    fi
    echo $i
done
```

---

## Functions

### Defining Functions

```bash
#!/bin/bash

# Method 1
function greet {
    echo "Hello, World!"
}

# Method 2 (preferred)
greet() {
    echo "Hello, World!"
}

# Call function
greet
```

### Functions with Arguments

```bash
#!/bin/bash

greet() {
    name=$1
    echo "Hello, $name!"
}

# Call with argument
greet "Alice"
greet "Bob"
```

### Functions with Return Values

```bash
#!/bin/bash

add() {
    result=$(($1 + $2))
    echo $result
}

# Capture return value
sum=$(add 5 3)
echo "Sum: $sum"
```

### Return Status

```bash
#!/bin/bash

check_file() {
    if [ -f "$1" ]; then
        return 0  # Success
    else
        return 1  # Failure
    fi
}

# Use return status
if check_file "test.txt"; then
    echo "File exists"
else
    echo "File not found"
fi
```

### Local Variables

```bash
#!/bin/bash

global_var="I am global"

my_function() {
    local local_var="I am local"
    echo "$global_var"
    echo "$local_var"
}

my_function
echo "$global_var"
# echo "$local_var"  # This would fail - variable not defined
```

---

## Output Redirection

### Standard Streams

```
0 - stdin  (standard input)
1 - stdout (standard output)
2 - stderr (standard error)
```

### Basic Redirection

```bash
# Redirect stdout to file (overwrite)
command > output.txt
echo "Hello" > file.txt

# Redirect stdout to file (append)
command >> output.txt
echo "World" >> file.txt

# Redirect stderr to file
command 2> error.txt

# Redirect both stdout and stderr
command > output.txt 2>&1
command &> output.txt  # Shorthand

# Redirect to separate files
command > output.txt 2> error.txt
```

### Examples

```bash
#!/bin/bash

# Save command output
ls -la > files.txt

# Append to log
echo "Backup started: $(date)" >> backup.log

# Discard errors
find / -name "*.conf" 2> /dev/null

# Log everything
./script.sh > output.log 2>&1

# Separate output and errors
./script.sh > success.log 2> error.log
```

### Here Documents (Heredoc)

```bash
#!/bin/bash

# Multi-line input
cat << EOF
This is line 1
This is line 2
This is line 3
EOF

# Create file with heredoc
cat << EOF > config.txt
server=localhost
port=8080
user=admin
EOF
```

---

## Exit Status

### Understanding Exit Codes

```bash
0       # Success
1-255   # Error (non-zero)
```

**Common exit codes:**
- `0` - Success
- `1` - General error
- `2` - Misuse of shell command
- `126` - Command cannot execute
- `127` - Command not found
- `130` - Script terminated by Ctrl+C

### Checking Exit Status

```bash
#!/bin/bash

# $? holds exit status of last command
ls /tmp
echo "Exit status: $?"  # 0 (success)

ls /nonexistent
echo "Exit status: $?"  # Non-zero (error)
```

### Using Exit Status

```bash
#!/bin/bash

# Check if command succeeded
if command; then
    echo "Command succeeded"
else
    echo "Command failed"
fi

# Alternative
command
if [ $? -eq 0 ]; then
    echo "Success"
else
    echo "Failed"
fi
```

### Setting Exit Status

```bash
#!/bin/bash

if [ $# -eq 0 ]; then
    echo "Error: No arguments provided"
    exit 1
fi

echo "Processing arguments..."
exit 0
```

---

## Practical Examples

### Example 1: System Information Script

```bash
#!/bin/bash

echo "========== System Information =========="
echo "Hostname: $(hostname)"
echo "OS: $(cat /etc/os-release | grep PRETTY_NAME | cut -d'"' -f2)"
echo "Kernel: $(uname -r)"
echo "Uptime: $(uptime -p)"
echo "CPU: $(lscpu | grep 'Model name' | cut -d':' -f2 | xargs)"
echo "Memory: $(free -h | awk 'NR==2 {print $2}')"
echo "Disk: $(df -h / | awk 'NR==2 {print $2}')"
echo "========================================"
```

### Example 2: Backup Script

```bash
#!/bin/bash

# Configuration
SOURCE="/home/user/documents"
BACKUP_DIR="/backup"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_FILE="backup-${DATE}.tar.gz"

# Create backup
echo "Starting backup..."
tar -czf "${BACKUP_DIR}/${BACKUP_FILE}" "$SOURCE"

if [ $? -eq 0 ]; then
    echo "Backup successful: ${BACKUP_FILE}"
else
    echo "Backup failed!"
    exit 1
fi
```

### Example 3: User Management

```bash
#!/bin/bash

# Check if running as root
if [ $EUID -ne 0 ]; then
    echo "This script must be run as root"
    exit 1
fi

read -p "Enter username to create: " username

# Check if user exists
if id "$username" &>/dev/null; then
    echo "User $username already exists"
    exit 1
fi

# Create user
useradd -m -s /bin/bash "$username"
passwd "$username"

echo "User $username created successfully"
```

### Example 4: Disk Usage Alert

```bash
#!/bin/bash

THRESHOLD=80

# Get disk usage percentage
USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')

if [ $USAGE -gt $THRESHOLD ]; then
    echo "WARNING: Disk usage is ${USAGE}% (threshold: ${THRESHOLD}%)"
    exit 1
else
    echo "Disk usage is normal: ${USAGE}%"
    exit 0
fi
```

### Example 5: Service Checker

```bash
#!/bin/bash

SERVICE="nginx"

if systemctl is-active --quiet "$SERVICE"; then
    echo "$SERVICE is running"
else
    echo "$SERVICE is not running. Starting..."
    systemctl start "$SERVICE"
    
    if systemctl is-active --quiet "$SERVICE"; then
        echo "$SERVICE started successfully"
    else
        echo "Failed to start $SERVICE"
        exit 1
    fi
fi
```

### Example 6: File Processor

```bash
#!/bin/bash

# Process all .log files in directory
for file in *.log
do
    # Check if file exists (handle empty glob)
    [ -f "$file" ] || continue
    
    echo "Processing $file..."
    
    # Count lines
    lines=$(wc -l < "$file")
    echo "  Lines: $lines"
    
    # Count errors
    errors=$(grep -c "ERROR" "$file")
    echo "  Errors: $errors"
    
    echo ""
done
```

---

## Best Practices

### Script Structure

```bash
#!/bin/bash

#######################################
# Description: What this script does
# Author: Your Name
# Date: 2024-01-15
#######################################

# Configuration
readonly CONFIG_FILE="/etc/myapp.conf"
readonly LOG_FILE="/var/log/myapp.log"

# Functions
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >> "$LOG_FILE"
}

main() {
    log "Script started"
    # Main logic here
    log "Script completed"
}

# Execute main function
main "$@"
```

### Common Tips

1. **Always quote variables:**
   ```bash
   "$variable"  # Good
   $variable    # Risky
   ```

2. **Check arguments:**
   ```bash
   if [ $# -ne 2 ]; then
       echo "Usage: $0 <arg1> <arg2>"
       exit 1
   fi
   ```

3. **Use meaningful variable names:**
   ```bash
   backup_dir="/backup"  # Good
   bd="/backup"          # Bad
   ```

4. **Comment your code:**
   ```bash
   # Check if file exists before processing
   if [ -f "$file" ]; then
       # Process file
   fi
   ```

5. **Use functions for repeated code:**
   ```bash
   check_status() {
       # Reusable code
   }
   ```

---

## Quick Reference

### Common Commands

```bash
echo        # Print text
read        # Read input
exit        # Exit script
return      # Return from function
sleep       # Pause execution
date        # Get date/time
whoami      # Current user
hostname    # System hostname
```

### Comparison Cheat Sheet

```bash
# Numeric
[ $a -eq $b ]   # Equal
[ $a -ne $b ]   # Not equal
[ $a -lt $b ]   # Less than
[ $a -gt $b ]   # Greater than

# String
[ "$a" = "$b" ]  # Equal
[ "$a" != "$b" ] # Not equal
[ -z "$a" ]      # Empty
[ -n "$a" ]      # Not empty

# File
[ -f file ]     # Regular file
[ -d dir ]      # Directory
[ -e path ]     # Exists
```

---

This comprehensive guide covers all the basics needed to start writing effective shell scripts!