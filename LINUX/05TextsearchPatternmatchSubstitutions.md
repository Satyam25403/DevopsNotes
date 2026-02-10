# Text Processing and Search Patterns

Comprehensive guide to powerful text processing tools in Linux: grep, sort, wc, uniq, awk, and sed.

## Table of Contents
- [grep - Global Regular Expression Print](#grep---global-regular-expression-print)
- [sort - Sort Lines](#sort---sort-lines)
- [wc - Word Count](#wc---word-count)
- [uniq - Unique Lines](#uniq---unique-lines)
- [awk - Pattern Scanning and Processing](#awk---pattern-scanning-and-processing)
- [sed - Stream Editor](#sed---stream-editor)
- [Practical Examples and Use Cases](#practical-examples-and-use-cases)

---

## grep - Global Regular Expression Print

Search for patterns in files and display matching lines.

### Basic Usage

```bash
grep <pattern> <filename>              # Search for exact word (matches substrings)
grep -w <pattern> <filename>           # Match whole word only
grep -i <pattern> <filename>           # Case-insensitive search
grep -r <pattern> <directory>          # Recursive search in directory
grep -c <pattern> <filename>           # Count matching lines
grep -n <pattern> <filename>           # Show line numbers
grep -v <pattern> <filename>           # Show lines NOT containing pattern
```

### Examples

```bash
# Basic search
grep "error" app.log                   # Find all lines with "error"

# Whole word match
grep -w "error" app.log                # Match "error" but not "errors" or "terrorize"

# Case insensitive
grep -i "ERROR" app.log                # Matches error, Error, ERROR, etc.

# Recursive search
grep -r "TODO" /path/to/project        # Search all files in project

# Count occurrences
grep -c "success" results.log          # Number of lines with "success"

# Show line numbers
grep -n "404" access.log               # Show where 404 errors occur

# Invert match (exclude)
grep -v "DEBUG" app.log                # Show all lines except DEBUG
```

### Using Regular Expressions

```bash
# Extended regex mode
grep -E "regex" <filename>

# Examples
grep -E "desktop|error" app.log        # Match "desktop" OR "error"
grep -E "^ERROR" app.log               # Lines starting with ERROR
grep -E "[0-9]{3}" file.txt            # Three consecutive digits
```

### Real-time Log Monitoring

```bash
# Follow logs and filter
tail -f app.log | grep -i error        # Watch for errors in real-time
tail -f access.log | grep "404"        # Monitor 404 errors live
```

### Common Options Combinations

```bash
grep -rni "error" /var/log             # Recursive, line numbers, case-insensitive
grep -rl "TODO" .                      # List files containing TODO
grep -A 3 "error" app.log              # Show 3 lines after match
grep -B 2 "error" app.log              # Show 2 lines before match
grep -C 2 "error" app.log              # Show 2 lines before and after
```

---

## sort - Sort Lines

Sort lines of text files alphabetically or numerically.

### Basic Usage

```bash
sort <filename>                        # Sort alphabetically (ascending)
sort -r <filename>                     # Reverse sort (descending)
sort -n <filename>                     # Numeric sort
sort -u <filename>                     # Sort and remove duplicates
```

### Examples

```bash
# Alphabetical sort
sort names.txt

# Reverse order
sort -r names.txt

# Numeric sort (important for numbers!)
sort -n numbers.txt
# Without -n: 1, 10, 2, 20 (wrong!)
# With -n: 1, 2, 10, 20 (correct!)

# Remove duplicates while sorting
sort -u emails.txt

# Sort by column
sort -k2 data.txt                      # Sort by 2nd column
sort -t: -k3 -n /etc/passwd           # Sort passwd by UID (-t: sets delimiter)
```

### Advanced Sorting

```bash
# Sort by multiple columns
sort -k1,1 -k2n data.txt              # Sort by col 1 (text), then col 2 (numeric)

# Reverse numeric sort
sort -rn numbers.txt

# Human-readable sizes
sort -h sizes.txt                      # Sorts 1K, 2M, 3G correctly

# Check if file is sorted
sort -c file.txt                       # Returns error if not sorted
```

---

## wc - Word Count

Count lines, words, and characters in files.

### Basic Usage

```bash
wc <filename>                          # Show lines, words, characters
wc -l <filename>                       # Count lines only
wc -w <filename>                       # Count words only
wc -c <filename>                       # Count bytes/characters only
```

### Examples

```bash
# Full count
wc file.txt
# Output: 100 500 3000 file.txt
#         │   │   │
#         │   │   └─ characters
#         │   └───── words
#         └───────── lines

# Count lines (most common)
wc -l app.log                          # How many log entries?

# Count files in directory
ls | wc -l                             # Number of files

# Count matching lines
grep "error" app.log | wc -l          # How many errors?

# Multiple files
wc -l *.txt                            # Line count for each .txt file
```

---

## uniq - Unique Lines

Remove or count duplicate adjacent lines.

### Important Concept

⚠️ **Critical:** `uniq` only removes **adjacent** duplicates!

**Wrong approach:**
```bash
uniq file.txt                          # Only removes consecutive duplicates
```

**Correct approach:**
```bash
sort file.txt | uniq                   # Sort first, then remove duplicates
```

### Basic Usage

```bash
uniq <filename>                        # Remove adjacent duplicates
uniq -c <filename>                     # Count occurrences
uniq -d <filename>                     # Show only duplicates
uniq -u <filename>                     # Show only unique lines
```

### Examples

```bash
# Remove duplicates (proper way)
sort file.txt | uniq

# Count occurrences
sort access.log | uniq -c | sort -rn   # Most frequent entries first

# Show only duplicates
sort emails.txt | uniq -d

# Show only unique lines
sort data.txt | uniq -u

# Case-insensitive comparison
sort -f file.txt | uniq -i
```

### Practical Use Cases

```bash
# Find unique IP addresses in access log
awk '{print $1}' access.log | sort | uniq

# Count unique visitors
awk '{print $1}' access.log | sort | uniq | wc -l

# Top 10 IP addresses
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

---

## awk - Pattern Scanning and Processing

Powerful text processing language for column-based filtering, aggregation, and reporting.

### Core Concepts

**How awk works:**
- Processes files line by line
- Splits each line into fields (columns)
- Applies patterns and actions

**Basic Syntax:**
```bash
awk 'pattern { action }' filename
```

### Built-in Variables

| Variable | Meaning |
|----------|---------|
| `$0` | Entire line |
| `$1` | First field/column |
| `$2` | Second field/column |
| `$NF` | Last field |
| `NF` | Number of fields in current line |
| `NR` | Current line number |

**Default field separator:** Space or tab

### Basic Usage

```bash
# Print specific columns
awk '{print $1}' file.txt              # Print first column
awk '{print $1, $3}' file.txt          # Print columns 1 and 3
awk '{print $NF}' file.txt             # Print last column
awk '{print NR, $0}' file.txt          # Print line number and entire line
```

### Custom Field Separator

```bash
# Change field separator
awk -F',' '{print $1, $3}' data.csv    # Use comma as separator
awk -F: '{print $1, $7}' /etc/passwd   # Use colon (user, shell)
awk -F'\t' '{print $2}' data.tsv       # Tab-separated
```

### Pattern Matching

```bash
# Print lines matching pattern
awk '/error/ {print $0}' app.log       # Lines containing "error"
awk '/^ERROR/ {print $1, $3}' app.log  # Lines starting with ERROR

# Conditional printing
awk '$3 > 100 {print $1, $3}' data.txt # Column 3 > 100
awk 'NR > 1 {print}' file.txt          # Skip header line
awk 'NF > 0' file.txt                  # Skip empty lines
```

### Built-in Functions

```bash
# String functions
awk '{print length($0)}' file.txt      # Length of each line
awk '{print toupper($1)}' file.txt     # Convert to uppercase
awk '{print tolower($1)}' file.txt     # Convert to lowercase
awk '{print substr($1,1,4)}' file.txt  # First 4 characters
```

### BEGIN and END Blocks

```bash
# Three-block structure
awk 'BEGIN { actions } pattern { actions } END { actions }' file
```

**Block execution:**
1. `BEGIN` - Runs once before reading file (initialization, headers)
2. Main block - Runs for every line
3. `END` - Runs once after all lines (summaries, totals)

### Examples with Blocks

```bash
# Simple header and footer
awk 'BEGIN {print "START"} {print $1} END {print "END"}' file.txt

# Output:
# START
# col1_line1
# col1_line2
# END

# Sum column
awk '{sum += $2} END {print sum}' data.txt

# Average calculation
awk '{sum += $2; count++} END {print sum/count}' data.txt

# Pretty output
awk 'BEGIN {print "Name\tScore"} {print $1, $2} END {print "---"}' scores.txt
```

### Advanced Examples

```bash
# Count occurrences
awk '{count[$1]++} END {for (ip in count) print ip, count[ip]}' access.log

# Sum by category
awk '{sum[$1] += $2} END {for (cat in sum) print cat, sum[cat]}' data.txt

# Filter and process
awk '$3 > 1000 {total += $3; count++} END {print total/count}' sales.txt

# Multiple conditions
awk '$2 == "ERROR" && $3 > 100 {print $1, $4}' app.log

# Format output
awk '{printf "%-10s %5d\n", $1, $2}' data.txt
```

### Why awk is Powerful (DevOps Perspective)

✅ **Advantages:**
- No temporary files needed
- Single-pass processing
- Handles huge logs efficiently
- Built-in math and string operations

**Common DevOps Uses:**
- Log analysis
- Monitoring
- CI/CD pipelines
- Reporting and metrics
- Data extraction and transformation

---

## sed - Stream Editor

Edit text streams (files, command output, piped input) using pattern matching and substitution.

### Core Concept

⚠️ **By default:** sed does NOT modify the file - it prints modified output to stdout.

**Basic Syntax:**
```bash
sed 'command' filename
```

### Substitution (Most Common)

```bash
# Basic substitution
sed 's/old/new/' file.txt              # Replace first occurrence per line
sed 's/old/new/g' file.txt             # Replace all occurrences per line (global)
sed 's/old/new/gi' file.txt            # Case-insensitive replacement
sed 's/old/new/2' file.txt             # Replace only 2nd occurrence per line
```

### Line-Specific Operations

```bash
# Operate on specific lines
sed '5s/old/new/' file.txt             # Replace only on line 5
sed '3,10s/foo/bar/g' file.txt         # Replace in lines 3-10
sed '1,5s/old/new/g' file.txt          # Replace in first 5 lines
sed '/pattern/s/old/new/g' file.txt    # Replace only in lines matching pattern
```

### In-place Editing

```bash
# Modify file directly
sed -i 's/old/new/g' file.txt          # Direct modification ⚠️

# Create backup before editing (safer!)
sed -i.bak 's/old/new/g' file.txt      # Creates file.txt.bak
```

**Use cases:**
```bash
# Change port in config
sed -i 's/8080/9090/g' app.conf

# Update environment variable
sed -i "s/DB_HOST=.*/DB_HOST=$DB_HOST/" .env

# Production-safe editing
sed -i.bak 's/old/new/g' config.txt    # Always use backup in production!
```

### Printing Lines

```bash
# Print specific lines
sed -n '5p' file.txt                   # Print line 5 only (-n suppresses other lines)
sed -n '5,10p' file.txt                # Print lines 5-10
sed -n '/pattern/p' file.txt           # Print lines matching pattern
sed -n '1p;$p' file.txt                # Print first and last line
```

### Deleting Lines

```bash
# Delete lines
sed '5d' file.txt                      # Delete line 5
sed '5,10d' file.txt                   # Delete lines 5-10
sed '/ERROR/d' app.log                 # Delete lines containing ERROR
sed '/^$/d' file.txt                   # Delete empty lines
sed '/^#/d' config.txt                 # Delete comment lines
```

### Inserting and Appending

```bash
# Insert before line
sed '5i This is new line' file.txt     # Insert before line 5

# Append after line
sed '5a This is new line' file.txt     # Append after line 5

# Insert at beginning
sed '1i Header Line' file.txt

# Append at end
sed '$a Footer Line' file.txt
```

### Using Regular Expressions

```bash
# Replace with regex
sed 's/[0-9]\+/NUMBER/g' file.txt      # Replace numbers with "NUMBER"
sed 's/^[ \t]*//' file.txt             # Remove leading whitespace
sed 's/[ \t]*$//' file.txt             # Remove trailing whitespace

# DevOps examples
sed -i "s/DB_HOST=.*/DB_HOST=$DB_HOST/" .env  # Replace env var
sed 's/ERROR\|FATAL/ALERT/g' app.log          # Replace ERROR or FATAL
```

### Multiple Commands

```bash
# Multiple sed commands
sed -e 's/foo/bar/g' -e 's/old/new/g' file.txt

# Alternative syntax
sed 's/foo/bar/g; s/old/new/g' file.txt

# Complex transformations
sed -e 's/^/PREFIX-/' -e 's/$/SUFFIX/' file.txt  # Add prefix and suffix
```

### Advanced Examples

```bash
# Remove comments and empty lines
sed -e '/^#/d' -e '/^$/d' config.txt

# Extract between markers
sed -n '/START/,/END/p' file.txt

# Change multiple values in config
sed -i \
  -e 's/PORT=.*/PORT=8080/' \
  -e 's/HOST=.*/HOST=localhost/' \
  config.ini

# Format log file
sed 's/\([0-9]\{4\}\)-\([0-9]\{2\}\)-\([0-9]\{2\}\)/\3\/\2\/\1/' dates.log
```

---

## Practical Examples and Use Cases

### Log Analysis Workflow

```bash
# Find errors, count by type
grep -E "ERROR|FATAL" app.log | \
  awk '{print $4}' | \
  sort | uniq -c | sort -rn

# Top 10 error messages
grep ERROR app.log | \
  awk '{$1=$2=$3=""; print $0}' | \
  sort | uniq -c | sort -rn | head -10

# Analyze access log
awk '{print $1}' access.log | \
  sort | uniq -c | sort -rn | \
  head -10                              # Top 10 IP addresses
```

### Data Processing Pipeline

```bash
# Extract, transform, load
cat data.csv | \
  sed '1d' | \                          # Remove header
  awk -F',' '$3 > 1000 {print $1, $2}' | \  # Filter
  sort -k2 -rn | \                      # Sort by 2nd column
  head -20                              # Top 20

# Clean and deduplicate
cat emails.txt | \
  sed 's/^[ \t]*//' | \                 # Trim whitespace
  sed 's/[ \t]*$//' | \
  tr '[:upper:]' '[:lower:]' | \        # Lowercase
  sort | uniq                           # Deduplicate
```

### Configuration Management

```bash
# Update multiple config values
sed -i.bak \
  -e 's/DEBUG=true/DEBUG=false/' \
  -e 's/PORT=3000/PORT=8080/' \
  -e 's/HOST=localhost/HOST=0.0.0.0/' \
  app.conf

# Replace variables in template
DB_HOST="prod-db.example.com"
DB_PORT="5432"
sed -e "s/\${DB_HOST}/$DB_HOST/g" \
    -e "s/\${DB_PORT}/$DB_PORT/g" \
    template.conf > production.conf
```

### Reporting and Metrics

```bash
# Generate summary report
awk 'BEGIN {print "=== Daily Report ==="} \
     /ERROR/ {errors++} \
     /SUCCESS/ {success++} \
     END {
       print "Errors:", errors
       print "Success:", success
       print "Success Rate:", (success/(errors+success)*100)"%"
     }' app.log

# Calculate statistics
awk '{sum+=$1; sumsq+=$1*$1} \
     END {
       print "Count:", NR
       print "Average:", sum/NR
       print "StdDev:", sqrt(sumsq/NR - (sum/NR)^2)
     }' numbers.txt
```

---

## Command Comparison and When to Use

| Tool | Best For | Speed | Complexity |
|------|----------|-------|------------|
| `grep` | Finding lines | Fast | Simple |
| `sed` | Text substitution | Fast | Medium |
| `awk` | Column processing | Fast | Medium-High |
| `sort` | Ordering data | Medium | Simple |
| `uniq` | Removing duplicates | Fast | Simple |

**General Guidelines:**
- **grep:** When you need to find/filter lines
- **sed:** When you need to replace/transform text
- **awk:** When you need column-based processing or calculations
- **Combine them:** Most powerful when piped together!

---

## Best Practices

1. **Test before modifying files:**
   - Test sed commands without `-i` first
   - Use `-i.bak` for backups in production

2. **Use appropriate tools:**
   - Don't use sed for column processing (use awk)
   - Don't use awk for simple line filtering (use grep)

3. **Optimize pipelines:**
   - Filter early (grep first to reduce data)
   - Minimize intermediate steps

4. **Handle edge cases:**
   - Empty lines
   - Special characters
   - Large files

5. **Performance tips:**
   - Use `-F` flag in grep for fixed strings (faster than regex)
   - Use awk for multiple operations instead of multiple pipes
   - Sort before uniq

---

## Quick Reference Cheat Sheet

```bash
# Search
grep -rni "pattern" dir/                # Recursive, line numbers, case-insensitive
grep -A 2 -B 2 "error" file            # Context: 2 lines before and after

# Sort
sort -k2,2n -k1,1 file                 # Multi-column sort
sort -t: -k3 -n /etc/passwd           # Custom delimiter

# Count
wc -l file                             # Line count (most common)
grep "pattern" file | wc -l            # Count matches

# Deduplicate
sort file | uniq                       # Remove duplicates
sort file | uniq -c | sort -rn        # Count and sort by frequency

# awk patterns
awk '{print $1, $NF}' file            # First and last column
awk -F: '/pattern/ {print $1}' file   # Custom delimiter + pattern
awk '{sum+=$2} END {print sum}' file  # Sum column

# sed transformations
sed 's/old/new/g' file                # Replace all
sed -i.bak 's/old/new/g' file         # In-place with backup
sed -n '10,20p' file                  # Print lines 10-20
sed '/pattern/d' file                 # Delete matching lines
```

This comprehensive guide covers the essential text processing tools every Linux administrator and DevOps engineer should master!