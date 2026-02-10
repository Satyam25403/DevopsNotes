# Filesystem and Operations

Comprehensive guide to Linux file operations, text processing, permissions, archiving, and advanced filesystem concepts.

## Table of Contents
- [File Creation and Viewing](#file-creation-and-viewing)
- [Text Processing and Editing](#text-processing-and-editing)
- [File Operations and Access Control](#file-operations-and-access-control)
- [Directory Operations](#directory-operations)
- [Navigation and Listing](#navigation-and-listing)
- [Command Chaining and Background Execution](#command-chaining-and-background-execution)
- [I/O Redirection and Error Handling](#io-redirection-and-error-handling)
- [Comparison Operators](#comparison-operators)
- [Searching Files and Directories](#searching-files-and-directories)
- [Archiving and Compression](#archiving-and-compression)
- [Inodes and Links](#inodes-and-links)

---

## File Creation and Viewing

### Creating Files with echo

```bash
echo "sample text"                     # Print to terminal
echo "file content" > filename         # Create/overwrite file
echo "file content" >> filename        # Append to file
```

**Examples:**
```bash
echo "Hello World" > greeting.txt      # Creates new file
echo "Line 2" >> greeting.txt          # Adds to existing file
```

---

## Text Processing and Editing

### Viewing File Contents

```bash
cat filename                           # Display entire file
cat -n filename                        # Show line numbers
cat file1 file2                        # Display multiple files
cat file1 file2 > combined             # Merge files
```

### Interactive Viewing

```bash
less filename                          # Scroll through file (line by line)
                                       # Use arrows, PgUp/PgDn, q to quit
more filename                          # Page-by-page viewing (older tool)
```

### Viewing Portions of Files

```bash
# View first N lines
head -n 10 file                        # First 10 lines
head -n 5 file1 file2                  # First 5 lines of multiple files

# View last N lines
tail -n 10 file                        # Last 10 lines
tail -f /var/log/syslog               # Follow file in real-time (for logs)
tail -n 20 file1 file2                # Last 20 lines of multiple files
```

---

## File Operations and Access Control

### Creating and Editing Files

```bash
touch filename.ext                     # Create empty file
vi filename                            # Edit with vi
vim filename                           # Edit with vim (improved vi)
nano filename                          # Edit with nano (beginner-friendly)
```

#### Basic vi/vim Commands

| Command | Action |
|---------|--------|
| `i` | Enter insert mode |
| `Esc` | Exit insert mode |
| `:w` | Save file |
| `:q` | Quit |
| `:wq` | Save and quit |
| `:q!` | Quit without saving |

### Copying and Moving Files

```bash
cp source target                       # Copy file
cp source1 source2 target_dir/         # Copy multiple files to directory
cp -r source_dir/ target_dir/          # Copy directory recursively
cp -p file1 file2                      # Preserve permissions and timestamps

mv source target                       # Move or rename file
mv file1 file2 dir/                    # Move multiple files to directory
```

### File Permissions

#### Understanding Permissions

```
-rwxr-xr--
│││││││││
│││││││└└─── Others: read
││││││└───── Others: execute (no)
│││││└────── Group: read
││││└─────── Group: execute
│││└──────── Group: write (no)
││└───────── Owner: read
│└────────── Owner: write
└─────────── Owner: execute
```

**Numeric values:**
- Read (r) = 4
- Write (w) = 2
- Execute (x) = 1

#### chmod - Change Permissions

```bash
# Numeric mode
chmod 755 filename                     # rwxr-xr-x
chmod 644 filename                     # rw-r--r--
chmod -R 755 directory                 # Recursive (all files/subdirs)

# Symbolic mode
chmod u+x script.sh                    # Add execute for owner (user)
chmod g+w file.txt                     # Add write for group
chmod o-r file.txt                     # Remove read for others
chmod a+r file.txt                     # Add read for all (user+group+others)
```

**Common permission patterns:**

| Numeric | Symbolic | Meaning | Use Case |
|---------|----------|---------|----------|
| `755` | `rwxr-xr-x` | Owner: full, Others: read+execute | Scripts, directories |
| `644` | `rw-r--r--` | Owner: read+write, Others: read | Regular files |
| `600` | `rw-------` | Owner only | Private files, SSH keys |
| `700` | `rwx------` | Owner only, full access | Private directories |

⚠️ **Note:** New files created later won't inherit permissions set with `chmod -R`. You need to rerun the command or use `umask`.

#### chown - Change Ownership

```bash
sudo chown username file               # Change owner
sudo chown username:groupname file     # Change owner and group
sudo chown -R username directory       # Recursive ownership change
```

### Default Permissions (umask)

`umask` defines the default permission mask for new files and directories. It **removes** permissions from system defaults.

#### Understanding umask

**Default permissions before umask:**

| Type | Default |
|------|---------|
| Files | `666` (rw-rw-rw-) |
| Directories | `777` (rwxrwxrwx) |

⚠️ **Important:** Files never get execute permission by default. You must set it explicitly.

**Example:**
```bash
umask                                  # View current umask (e.g., 0022)
umask 002                              # Set umask for current session
```

**umask = 002 means:**
- Owner → Remove nothing
- Group → Remove nothing
- Others → Remove write (2)

**Result:**
- New files: `664` (rw-rw-r--)
- New directories: `775` (rwxrwxr-x)

#### Setting umask Persistently

**Per user:**
Add to shell config files:
- `~/.bashrc`
- `~/.bash_profile`
- `~/.zshrc`

**System-wide:**
Edit:
- `/etc/profile`
- `/etc/bashrc`
- `/etc/login.defs`

Example in `/etc/login.defs`:
```bash
UMASK 002
```

⚠️ **Note:** 
- Terminal sessions have their own umask
- Systemd services don't inherit shell's umask
- Different users can have different umask values

### SetGID Bit for Directories

Ensure new files inherit the directory's group ownership.

```bash
chmod g+s directory                    # Set setgid bit
ls -ld directory                       # Verify (look for 's' in group permissions)
```

**Example output:**
```
drwxrws--- 2 hanks developers 4096 Jan 15 10:30 directory
        ↑
        setgid bit
```

**Effect:** Any new file or directory created inside automatically inherits the directory's group.

---

## Directory Operations

```bash
mkdir directory                        # Create directory
mkdir -p /path/to/nested/dir          # Create nested directories
rmdir directory                        # Remove empty directory only
rm -rf directory                       # Recursive force removal ⚠️ Dangerous!
rm -r -i directory                     # Interactive deletion (prompts before each)
```

---

## Navigation and Listing

```bash
cd /path/to/directory                  # Change directory
cd ~                                   # Go to home directory
cd ..                                  # Go up one level
cd -                                   # Go to previous directory

pwd                                    # Print working directory

ls                                     # List files
ls -l                                  # Long format (permissions, owner, size)
ls -a                                  # Include hidden files (starting with .)
ls -R                                  # Recursive listing
ls -lah                                # Common: long, all, human-readable
ls -lart                              # Long, all, reverse time order
```

---

## Command Chaining and Background Execution

### Command Chaining

```bash
command1 && command2                   # Run command2 only if command1 succeeds
command1 || command2                   # Run command2 only if command1 fails
command1 ; command2                    # Run both regardless of success
```

**Examples:**
```bash
mkdir backup && cp *.txt backup/       # Create dir, then copy if successful
rm file.txt || echo "Delete failed"    # Try delete, show message if fails
cd /tmp ; ls                           # Change dir and list (both run)
```

### Background Execution

```bash
command &                              # Run in background
nohup command &                        # Run immune to logout/hangup
                                       # (continues after you log out)
```

**Examples:**
```bash
tar -czf backup.tar.gz /data &         # Compress in background
nohup python3 server.py > output.log 2>&1 &  # Server survives logout
```

---

## I/O Redirection and Error Handling

### Redirecting Output and Errors

```bash
command > output.log                   # Redirect stdout, overwrite
command >> output.log                  # Redirect stdout, append
command 2> error.log                   # Redirect stderr only
command > output.log 2>&1              # Redirect both stdout and stderr to same file
command >> output.log 2>&1             # Append both to same file
command 2> /dev/null                   # Discard errors
command 1> /dev/null 2> error.log      # Discard output, log only errors
command &> all.log                     # Redirect both (shorthand)
```

**File descriptors:**
- `0` - stdin (standard input)
- `1` - stdout (standard output)
- `2` - stderr (standard error)

### Examples

```bash
# Find files, ignore permission errors
find / -name "*.conf" 2> /dev/null

# Log both output and errors
script.sh > output.log 2>&1

# Separate output and error logs
command > output.log 2> error.log

# Discard all output
command > /dev/null 2>&1
```

---

## Comparison Operators

### For Numbers

```bash
-lt     # Less than
-gt     # Greater than
-le     # Less than or equal
-ge     # Greater than or equal
-eq     # Equal
-ne     # Not equal
```

**Example:**
```bash
if [ $age -gt 18 ]; then
    echo "Adult"
fi
```

### For Strings

```bash
==      # Equal
!=      # Not equal
-z      # String is empty
-n      # String is not empty
```

**Example:**
```bash
if [ "$name" == "admin" ]; then
    echo "Welcome admin"
fi
```

---

## Searching Files and Directories

### find - Search Filesystem

```bash
find /path -name "filename"            # Search by name
find . -name "*.txt"                   # Search current dir for .txt files
find /var -type f                      # Find files only
find /var -type d                      # Find directories only
find / -perm 755                       # Find by permissions
find /home -mtime -7                   # Modified in last 7 days
find . -size +100M                     # Files larger than 100MB
find . -user username                  # Files owned by user
```

**Advanced examples:**
```bash
# Find and delete old log files
find /var/log -name "*.log" -mtime +30 -delete

# Find and execute command
find . -name "*.txt" -exec grep "error" {} \;

# Find large files
find / -type f -size +1G 2>/dev/null
```

### locate - Fast File Search

Uses a database (updated daily by `updatedb`).

```bash
locate filename                        # Fast search
locate -i "*.log"                      # Case-insensitive
sudo updatedb                          # Update locate database
```

**Differences:**
- `find` - Real-time, slower, more features
- `locate` - Database-based, faster, may be outdated

---

## Archiving and Compression

### Concepts

- **Archive:** Combines files/directories into one file (no size reduction)
  - Tool: `tar`
- **Compression:** Reduces file size using algorithms
  - Tools: `gzip`, `bzip2`, `xz`, `zip`

### tar - Archiving (Tape ARchiver)

```bash
# Create archive
tar -cvf archive.tar file1 file2 dir/  # Create archive
# c = create, v = verbose, f = filename

# Extract archive
tar -xvf archive.tar                   # Extract all
tar -xvf archive.tar -C /target/dir    # Extract to specific directory

# List contents
tar -tvf archive.tar                   # View contents without extracting
```

### Compression Tools (Single Files)

#### gzip (fast)

```bash
gzip file.txt              # Creates file.txt.gz (original removed)
gunzip file.txt.gz         # Decompress
gzip -k file.txt           # Keep original file
```

#### bzip2 (better compression)

```bash
bzip2 file.txt             # Creates file.txt.bz2
bunzip2 file.txt.bz2       # Decompress
```

#### xz (best compression)

```bash
xz file.txt                # Creates file.txt.xz
unxz file.txt.xz           # Decompress
```

### tar + Compression Combined

#### tar.gz (most common)

```bash
# Create compressed archive
tar -czvf backup.tar.gz /directory     # -z for gzip
# OR
tar -czf backup.tgz /directory         # .tgz is shorthand for .tar.gz

# Extract
tar -xzvf backup.tar.gz
```

#### tar.bz2 (better compression)

```bash
# Create
tar -cjvf backup.tar.bz2 /directory    # -j for bzip2

# Extract
tar -xjvf backup.tar.bz2
```

#### tar.xz (best compression)

```bash
# Create
tar -cJvf backup.tar.xz /directory     # -J for xz

# Extract
tar -xJvf backup.tar.xz
```

### zip - Cross-platform Archive

```bash
# Create zip
zip archive.zip file1.txt file2.txt    # Originals remain
zip -r directory.zip /directory        # Recursive directory zip
zip -e secure.zip file.txt             # Encrypted zip (prompts for password)

# Extract
unzip archive.zip                      # Extract to current dir
unzip archive.zip -d /target/dir       # Extract to specific directory
```

### Compression Comparison

| Format | Compression | Speed | Use Case |
|--------|-------------|-------|----------|
| `tar` | ❌ None | Fast | Just bundle files |
| `tar.gz` / `.tgz` | ✅ Good | Fast | Daily backups, most common |
| `tar.bz2` | ✅ Better | Medium | Better compression needed |
| `tar.xz` | ✅ Best | Slow | Maximum compression |
| `zip` | ✅ Good | Medium | Cross-platform compatibility |

### Common Use Cases

```bash
# Daily backup (fast)
tar -czf backup-$(date +%Y%m%d).tar.gz /home/user

# Maximum compression for archival
tar -cJf archive.tar.xz /important/data

# Cross-platform sharing
zip -r project.zip /project

# Encrypted backup
zip -e -r secure.zip /sensitive/data
```

---

## Inodes and Links

### Understanding Inodes

An **inode** (index node) is a data structure that stores metadata about a file.

**Inode contains:**
- File size
- Permissions
- Owner and group
- Timestamps (created, modified, accessed)
- Number of hard links
- Pointers to data blocks

**Inode does NOT contain:**
- Filename
- File data itself

### How File Access Works

```bash
cat file.txt
```

**Process:**
1. Linux looks up `file.txt` in directory
2. Finds inode number
3. Uses inode to locate data blocks on disk
4. Reads file content

### Viewing Inodes

```bash
ls -li                                 # Show inode numbers
df -i                                  # Check inode usage
stat filename                          # Detailed inode information
```

**Example output:**
```
123456 -rw-r--r-- 1 user group 1024 Jan 15 10:30 file.txt
  ↑
inode number
```

### Inode Exhaustion Problem

Classic DevOps issue:
- ❌ Free disk space available
- ❌ But cannot create files

**Cause:** All inodes are used (too many small files)

**Diagnosis:**
```bash
df -h           # Shows free space ✅
df -i           # Shows 100% inode usage ⚠️
```

**Solution:**
- Delete unnecessary files
- Restructure storage (combine small files)

---

## Links (Hard Links and Symbolic Links)

A link is another name or reference to a file.

### Hard Links

A hard link points directly to the file's inode.

**Characteristics:**
- Multiple filenames → Same inode → Same data
- No "original vs copy" concept
- File exists as long as at least one hard link exists
- All hard links are equal

```bash
ln original.txt hardlink.txt           # Create hard link
ls -li original.txt hardlink.txt       # Same inode number
```

**Example:**
```
123456 -rw-r--r-- 2 user group 1024 Jan 15 10:30 original.txt
123456 -rw-r--r-- 2 user group 1024 Jan 15 10:30 hardlink.txt
  ↑                  ↑
same inode      link count = 2
```

**Behavior:**
- Editing one updates both (same data)
- Deleting original → hard link still works
- File deleted only when link count = 0

**Limitations:**
- Cannot link directories
- Cannot cross filesystem boundaries

### Symbolic Links (Soft Links)

A symbolic link is a shortcut that stores the path to another file.

**Characteristics:**
- Has its own inode
- Stores path to target file
- If target deleted → broken symlink

```bash
ln -s original.txt symlink.txt         # Create symbolic link
ls -li original.txt symlink.txt        # Different inode numbers
```

**Example:**
```
123456 -rw-r--r-- 1 user group 1024 Jan 15 10:30 original.txt
789012 lrwxrwxrwx 1 user group   12 Jan 15 10:31 symlink.txt -> original.txt
  ↑                                                              ↑
different inode                                            points to path
```

**Behavior:**
- Points to file path (not inode)
- Can link directories
- Can cross filesystem boundaries
- Broken if target is moved or deleted

### Hard Link vs Symbolic Link

| Feature | Hard Link | Symbolic Link |
|---------|-----------|---------------|
| Points to | Inode directly | File path |
| Works after target deleted | ✅ Yes | ❌ No (broken) |
| Can link directories | ❌ No | ✅ Yes |
| Cross filesystems | ❌ No | ✅ Yes |
| Own inode | ❌ No | ✅ Yes |
| Use case | Backup, multiple names | Shortcuts, references |

### Practical Examples

```bash
# Hard link for backup
ln important.txt backup.txt
# If important.txt is deleted, backup.txt still has the data

# Symbolic link for shortcut
ln -s /var/www/html/project /home/user/project
# Easy access without duplicating data

# Check link type
ls -l symlink.txt
# Output starts with 'l' for symbolic link

# Find broken symbolic links
find /path -xtype l
```

---

## Quick Reference: File Operations

| Operation | Command | Example |
|-----------|---------|---------|
| Create file | `touch`, `echo >` | `touch file.txt` |
| View file | `cat`, `less` | `cat file.txt` |
| Edit file | `vi`, `nano` | `nano file.txt` |
| Copy | `cp` | `cp file.txt backup.txt` |
| Move/Rename | `mv` | `mv old.txt new.txt` |
| Delete | `rm` | `rm file.txt` |
| Change permissions | `chmod` | `chmod 755 script.sh` |
| Change owner | `chown` | `sudo chown user file.txt` |
| Search | `find`, `locate` | `find . -name "*.log"` |
| Archive | `tar` | `tar -czf backup.tar.gz /dir` |
| Compress | `gzip`, `zip` | `gzip file.txt` |

---

## Best Practices

1. **Always backup before making changes** to important files
2. **Use absolute paths** in scripts for reliability
3. **Test commands** on sample data before running on production
4. **Set appropriate permissions:**
   - Scripts: `755` (rwxr-xr-x)
   - Config files: `644` (rw-r--r--)
   - Private keys: `600` (rw-------)
5. **Use symbolic links** for shortcuts and references
6. **Use hard links** for backup and multiple names
7. **Monitor inode usage** (`df -i`) on systems with many small files
8. **Compress backups** to save space (`tar.gz` or `tar.xz`)
9. **Use `-i` flag** with `rm` for interactive deletion of important data
10. **Redirect errors** appropriately in scripts (`2>&1`)

---

This comprehensive guide covers essential filesystem operations and concepts for Linux system administration and DevOps work.