# Process Performance and Monitoring

Comprehensive guide to monitoring, managing, and controlling Linux processes, system resources, and scheduled tasks.

## Table of Contents
- [System Uptime](#system-uptime)
- [Interactive Process Monitoring](#interactive-process-monitoring)
- [Process Snapshot (Non-interactive)](#process-snapshot-non-interactive)
- [Process Identification and Control](#process-identification-and-control)
- [Process Priority](#process-priority-nice-and-renice)
- [Scheduled Tasks](#scheduled-tasks-cron-jobs)
- [Log Management](#log-management)
- [System Monitoring and Logging](#system-monitoring-and-logging-journalctl-systemd-dmesg)
- [System Resource Monitoring](#system-resource-monitoring)
- [Service Management](#service-management)
- [Background and Job Control](#background-and-job-control)

---

## System Uptime

```bash
uptime          # System uptime, users logged in, and load average
```

**Example output:**
```
10:23:45 up 5 days, 3:42, 2 users, load average: 0.52, 0.58, 0.61
```

---

## Interactive Process Monitoring

Real-time monitoring of running processes and CPU usage.

```bash
top                     # Real-time process monitoring
htop                    # Enhanced interactive process viewer (install: apt install htop)
top -p <pid>            # Monitor a specific process
```

### Key Metrics in top/htop

| Metric | Description |
|--------|-------------|
| `%CPU` | CPU usage per process |
| `us` | User processes CPU time |
| `sy` | System/kernel CPU time |
| `id` | Idle CPU percentage |
| `wa` | I/O wait ⚠️ **Very important** - high values indicate disk bottleneck |

---

## Process Snapshot (Non-interactive)

Get a snapshot of current processes without real-time updates.

### Basic ps Commands

```bash
ps                              # Processes running in current terminal
ps -a                           # All processes
ps -p <pid>                     # Info about a specific process
ps -u <username>                # Processes for a specific user
ps aux                          # All processes with detailed view (CPU, MEM, user, command)
```

### Advanced ps Usage

```bash
# Process tree view - very useful for debugging orphan/zombie processes
ps -ef --forest

# Sorted process listing
ps aux --sort=<field>

# Examples of sorting
ps aux --sort=-%cpu             # Descending CPU order (- means descending)
ps aux --sort=%mem              # Ascending memory order
ps aux --sort=-%cpu,-%mem       # Sort by CPU first, then memory if tie

# Using -eo (recommended in DevOps scripts)
ps -eo pid,user,%cpu,%mem,cmd --sort=-%cpu | head
```

### Common Sort Parameters
- `%cpu` - CPU usage
- `%mem` - Memory usage
- `start_time` - Process start time
- `pid` - Process ID
- `user` - Username

---

## Process Identification and Control

### Finding Processes

```bash
pidof <processname>             # Get PID(s) of a running process
                                # Example: pidof nginx

pgrep <processname>             # Find PIDs by name
```

### Killing Processes

```bash
kill <pid>                      # Send SIGTERM (graceful termination)
kill -15 <pid>                  # Same as above (SIGTERM)
kill -9 <pid>                   # Force kill (SIGKILL) - use as last resort

killall -9 <processname>        # Kill all instances of a process
                                # Example: multiple Chrome windows

pkill <processname>             # Kill processes by name
```

⚠️ **Important:** Always try `kill` (SIGTERM) before `kill -9` (SIGKILL) to allow graceful shutdown.

---

## Process Priority (nice and renice)

Process priority controls how much CPU time a process gets.

### Priority Values
- `-20` → Highest priority
- `0` → Default priority
- `19` → Lowest priority

**Higher nice value = Lower priority**

### Commands

```bash
# View current priorities
ps -o pid,ni,cmd                        # Display PID, nice value, command

# Start process with specific priority
nice -n <value> <command>               # Example: nice -n 10 tar -czf backup.tar.gz /data

# Change priority of running process
renice -n <value> <pid>                 # Needs root for increasing priority (lower nice value)
renice -n <value> -u <username>         # Change priority for all processes of a user
```

---

## Scheduled Tasks (Cron Jobs)

### One-Time Jobs (at)

Schedule tasks to run once at a specific time.

```bash
# Install at if missing
sudo apt install at

# Schedule a one-time job
echo "Hello Kali" | at 08:04
<command> | at <hh:mm>
<command> | at now + <t> minutes        # Example: at now + 30 minutes

# Manage at jobs
atq                                     # List pending jobs (shows job ID, time, owner)
atrm <jobid>                           # Delete a scheduled job
```

### Recurring Jobs (crontab)

Schedule tasks to run repeatedly.

#### Time Format
```
* * * * * command
│ │ │ │ │
│ │ │ │ └─── Day of week (0-7, Sun=0 or 7)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)
```

#### Time Symbols

| Symbol | Meaning |
|--------|---------|
| `*` | Every value |
| `*/5` | Every 5 units |
| `,` | Multiple values (e.g., `1,15,30`) |
| `-` | Range (e.g., `1-5`) |

#### Crontab Commands

```bash
crontab -l                              # List current user's cron jobs
crontab -e                              # Edit crontab (opens editor)
crontab -r                              # Delete all cron jobs ⚠️ Dangerous!
```

#### Examples

```bash
# Backup /home every day at 5:00 AM
0 5 * * * tar -czf /var/backups/home.tgz /home

# Run script every 15 minutes
*/15 * * * * /path/to/script.sh

# Run on weekdays at 6 PM
0 18 * * 1-5 /path/to/command

# Run first day of month
0 0 1 * * /path/to/monthly-task.sh
```

⚠️ **Note:** Scripts in cronjobs must be executable (`chmod +x script.sh`)

#### Viewing Cron Logs

```bash
journalctl -u cron                      # View cron service logs
cat /var/log/syslog | grep CRON        # Alternative log viewing
```

---

## Log Management

`logrotate` handles log file rotation, compression, and cleanup automatically.

### How logrotate Works

```
Application writes logs → /var/log/app.log
         ↓
logrotate checks rules
         ↓
Renames app.log → app.log.1
         ↓
Creates new app.log
         ↓
Compresses old logs
         ↓
Keeps N rotations
```

### Configuration

**Main config:** `/etc/logrotate.conf`
**Service configs:** `/etc/logrotate.d/`

Each service has its own config:
- `/etc/logrotate.d/nginx`
- `/etc/logrotate.d/syslog`

### Running logrotate

On modern systems, logrotate runs via:
- **Cron:** `/etc/cron.daily/logrotate`
- **Systemd timer:** `logrotate.timer`

```bash
# Manual run
sudo logrotate /etc/logrotate.conf

# Test configuration
sudo logrotate -d /etc/logrotate.conf   # Debug mode (dry run)
```

### Example logrotate Rule

```bash
/var/log/myapp.log {
    daily                   # Rotate daily
    rotate 7                # Keep 7 rotations
    compress                # Compress old logs
    missingok               # Don't error if log missing
    notifempty              # Don't rotate if empty
    create 0640 myapp myapp # Create new log with these permissions
}
```

⚠️ **Important:** logrotate never handles log content — only filenames, rotation, compression, and cleanup.

---

## System Monitoring and Logging (journalctl, systemd, dmesg)

Modern Linux systems use `systemd` and `journalctl` for centralized logging.

### journalctl - System Journal

View all system logs (kernel, service, system messages) sorted chronologically.

```bash
journalctl                              # All logs (old → new)
journalctl -u <servicename>             # Logs for specific service (e.g., nginx)
journalctl -f                           # Follow logs in real-time (Ctrl+C to stop)
journalctl -p <severity>                # Filter by severity level
```

#### Time-based Filtering

```bash
journalctl --since "2024-01-15"
journalctl --since "2024-01-15 10:30:00"
journalctl --since "1 hour ago"
journalctl --since "yesterday"
journalctl --until "2024-01-20"
journalctl --since "2024-01-01" --until "2024-01-31"
```

#### Severity Levels

| Level | Meaning |
|-------|---------|
| `emerg` | System unusable |
| `alert` | Immediate action needed |
| `crit` | Critical |
| `err` | Errors |
| `warning` | Warnings |
| `notice` | Normal but significant |
| `info` | Informational |
| `debug` | Debugging |

### systemd Unit Types

```bash
systemctl list-units --type=<unit>      # List active units of a type
```

| Unit Type | Purpose | Examples |
|-----------|---------|----------|
| `service` | Background services | docker, nginx, ssh |
| `socket` | Socket-activated services | ssh.socket, docker.socket |
| `target` | Logical grouping/runlevels | multi-user.target, graphical.target |
| `device` | Hardware devices | dev-sda.device |
| `mount` | Mounted filesystems | home.mount, var-log.mount |
| `automount` | On-demand mounts | |
| `path` | File/directory watchers | logrotate.path |
| `timer` | Scheduled jobs (cron alternative) | logrotate.timer |
| `slice` | Resource management | system.slice, user.slice |
| `scope` | External processes | SSH sessions |

⚠️ **Note:** `systemctl list-units` shows only system-level units, not user applications like Chrome.

### dmesg - Kernel Ring Buffer

View kernel messages since last boot (boot logs, hardware detection, driver messages).

```bash
dmesg                                   # Kernel messages
dmesg -T                                # With timestamps (human-readable)
dmesg -w                                # Follow kernel messages in real-time
dmesg | grep -i error                   # Filter for errors
```

**Key difference:**
- `journalctl` shows system + service logs
- `dmesg` shows kernel messages only

---

## System Resource Monitoring

### Memory

```bash
free -h                                 # Memory usage (human-readable)
```

**Example output:**
```
              total        used        free      shared  buff/cache   available
Mem:           15Gi       8.2Gi       1.3Gi       524Mi       6.1Gi       6.8Gi
Swap:         2.0Gi       128Mi       1.9Gi
```

### CPU and I/O Statistics

```bash
vmstat                                  # CPU, memory, swap, I/O statistics summary
vmstat 1                                # Update every 1 second

iostat                                  # Disk I/O performance (requires sysstat package)
iostat -x                               # Extended statistics
iostat -x 2                             # Update every 2 seconds

mpstat -P ALL                           # CPU usage per core
```

---

## Load and Performance

```bash
watch uptime                            # Continuously monitor load average
watch -n 1 "ps aux --sort=-%cpu | head" # Top CPU processes, updated every 1 second
```

### Understanding Load Average

Load average shows the number of processes waiting for CPU time.

**Interpretation:**
- Load ≈ number of CPU cores → OK
- Load > CPU cores → System stress
- Load >> CPU cores → System overloaded

Example: On a 4-core system:
- Load of 4.0 → Fully utilized
- Load of 8.0 → Overloaded (processes waiting)

---

## Service Management

Modern service control using `systemd`.

```bash
systemctl status <service>              # Check service health
systemctl start <service>               # Start service
systemctl stop <service>                # Stop service
systemctl restart <service>             # Restart service
systemctl enable <service>              # Start service on boot
systemctl disable <service>             # Prevent service from starting on boot
systemctl is-active <service>           # Check if service is running
systemctl is-enabled <service>          # Check if service starts on boot
```

⚠️ **Note:** Use `systemctl` instead of legacy `service` command on modern Linux.

---

## Process Tree and Hierarchy

```bash
pstree                                  # Visualize process tree
pstree -p                               # Show PIDs in tree
pstree -u                               # Show user transitions
```

---

## Background and Job Control

Shell-level process control.

```bash
jobs                                    # List background jobs
bg                                      # Resume job in background
fg                                      # Bring job to foreground
<command> &                             # Run command in background
nohup <command> &                       # Run process immune to logout/hangup
```

### Examples

```bash
# Start long-running process in background
tar -czf backup.tar.gz /data &

# Run process that survives logout
nohup python3 server.py > output.log 2>&1 &

# List background jobs
jobs
# Output: [1]+ Running    tar -czf backup.tar.gz /data &

# Bring to foreground
fg %1
```

---

## Quick Reference: Process Management Workflow

1. **Find processes:** `ps aux`, `pgrep`, `pidof`
2. **Monitor processes:** `top`, `htop`
3. **Control processes:** `kill`, `killall`, `pkill`
4. **Check resources:** `free -h`, `vmstat`, `iostat`
5. **View logs:** `journalctl`, `dmesg`
6. **Manage services:** `systemctl`
7. **Schedule tasks:** `crontab`, `at`

---

## Best Practices

1. **Always try graceful termination** (`kill` or `kill -15`) before force kill (`kill -9`)
2. **Monitor I/O wait** (`wa` in top) - high values indicate disk bottleneck
3. **Use systemctl** for service management on modern systems
4. **Check logs regularly** with `journalctl` for troubleshooting
5. **Set appropriate priorities** with `nice`/`renice` for background tasks
6. **Make scripts executable** before adding to cron
7. **Test cron jobs** manually before scheduling
8. **Rotate logs** to prevent disk space issues