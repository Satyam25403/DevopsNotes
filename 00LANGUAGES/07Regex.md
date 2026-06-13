# Regex — DevOps Reference Notes

> **Regular Expressions** — a pattern-matching language embedded in virtually every tool in the DevOps stack. In practice: **grep/egrep** for log triage, **sed** for in-place config transforms, **awk** for structured log parsing, **Nginx** rewrite rules, **CI filter patterns** (branch/path), **Prometheus** relabeling, **Alertmanager** routing, **Python/Go/Rust** for automation scripts, and **Kubernetes** admission webhooks. Regex dialects differ — this reference covers POSIX BRE/ERE, PCRE, RE2, and tool-specific variants with DevOps-relevant examples throughout.

---

## Table of Contents

1. [Regex Dialects & Where Each Is Used](#1-regex-dialects--where-each-is-used)
2. [Core Syntax — Anchors, Literals & Metacharacters](#2-core-syntax--anchors-literals--metacharacters)
3. [Character Classes & Shorthand](#3-character-classes--shorthand)
4. [Quantifiers](#4-quantifiers)
5. [Groups, Captures & Alternation](#5-groups-captures--alternation)
6. [Lookahead, Lookbehind & Assertions (PCRE)](#6-lookahead-lookbehind--assertions-pcre)
7. [Flags & Modes](#7-flags--modes)
8. [grep & egrep — Log Triage](#8-grep--egrep--log-triage)
9. [sed — Stream Editing & Config Transforms](#9-sed--stream-editing--config-transforms)
10. [awk — Structured Log Parsing](#10-awk--structured-log-parsing)
11. [Nginx — Regex in Location & Rewrite](#11-nginx--regex-in-location--rewrite)
12. [CI/CD — Branch, Path & Trigger Filters](#12-cicd--branch-path--trigger-filters)
13. [Prometheus — Relabeling & Alertmanager Routing](#13-prometheus--relabeling--alertmanager-routing)
14. [Python — re Module for Automation](#14-python--re-module-for-automation)
15. [Common DevOps Patterns](#15-common-devops-patterns)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Regex Dialects & Where Each Is Used

```
Dialect    Where used                              Key differences
─────────────────────────────────────────────────────────────────────────────
BRE        grep (default), sed (default)           ( ) { } + ? | need backslash
ERE        grep -E / egrep, awk, sed -E, POSIX     ( ) { } + ? | are metacharacters
PCRE       grep -P, nginx, Python re, Perl         Lookahead/behind, \d \w \s, named groups
RE2        Go, Kubernetes, Prometheus, re2          No lookahead/behind — safe linear time
JavaScript /pattern/flags                          No lookbehind in older engines
Java       java.util.regex                         PCRE-like, verbose mode
```

### Quick dialect comparison

```
Feature              BRE        ERE        PCRE       RE2
─────────────────────────────────────────────────────────
Groups               \( \)      ( )        ( )        ( )
Named groups         ✗          ✗          (?P<n>)    (?P<n>)
Non-capturing group  ✗          ✗          (?:)       (?:)
Alternation          \|         |          |          |
One-or-more          \+         +          +          +
Zero-or-one          \?         ?          ?          ?
Repetition           \{n,m\}    {n,m}      {n,m}      {n,m}
Lookahead            ✗          ✗          (?=)(?!)   ✗
Lookbehind           ✗          ✗          (?<=)(?<!) ✗
\d \w \s             ✗          ✗          ✓          ✓
Backreferences       \1         \1         \1         ✗
```

---

## 2. Core Syntax — Anchors, Literals & Metacharacters

### Anchors

```
^          start of string (or line in multiline mode)
$          end of string (or line in multiline mode)
\b         word boundary — between \w and \W (PCRE/ERE)
\B         non-word boundary
\A         start of entire string (PCRE — not affected by multiline)
\Z         end of entire string (PCRE)
\z         absolute end of string (PCRE)
```

```bash
# Anchor examples
grep '^ERROR'   app.log     # lines starting with ERROR
grep 'FATAL$'   app.log     # lines ending with FATAL
grep '^$'       app.log     # empty lines
grep -v '^$'    app.log     # non-empty lines
grep -v '^#'    config.conf # non-comment lines
grep '^ERROR\|^WARN' app.log  # BRE: lines starting with ERROR or WARN
grep -E '^(ERROR|WARN)' app.log  # ERE equivalent
```

### Metacharacters — must escape to match literally

```
.   *   +   ?   [   ]   {   }   (   )   ^   $   |   \
```

```bash
# To match a literal dot:
grep '\.'      file     # BRE/ERE — escape with backslash
grep -F '.'    file     # -F = fixed string, no regex at all (fastest)

# To match a literal dollar sign:
grep '\$'      file
grep '\$HOME'  file     # matches the string "$HOME"

# To match a literal backslash:
grep '\\\\'    file     # need 4 backslashes in shell → 2 in regex → matches \
```

### The dot — matches any character

```bash
grep 'c.t'     file     # cat, cut, cot, c9t, c t — any char between c and t
grep '^.\{8\}$' file    # BRE: exactly 8-char lines
grep -E '^.{8}$' file   # ERE equivalent
```

---

## 3. Character Classes & Shorthand

### Character classes `[...]`

```
[abc]      any one of: a, b, c
[^abc]     any character NOT a, b, or c
[a-z]      any lowercase letter
[A-Z]      any uppercase letter
[0-9]      any digit
[a-zA-Z]   any letter
[a-zA-Z0-9_]  word characters (manual)
[.-]       literal dot and hyphen (hyphen at start/end = literal)
```

```bash
grep '[Ee]rror'         app.log   # Error or error
grep '[0-9]\{3\}-[0-9]\{4\}' file  # BRE: phone pattern 555-1234
grep -E '[0-9]{3}-[0-9]{4}' file   # ERE
grep '[^a-zA-Z0-9]'     file   # lines with non-alphanumeric chars
grep '^[[:space:]]*$'   file   # blank / whitespace-only lines
```

### POSIX character classes (portable, work in BRE/ERE/awk)

```
[:alpha:]   letters [A-Za-z]
[:digit:]   digits [0-9]
[:alnum:]   letters + digits
[:space:]   whitespace (space, tab, newline, etc.)
[:blank:]   space and tab only
[:upper:]   uppercase letters
[:lower:]   lowercase letters
[:punct:]   punctuation
[:print:]   printable characters
[:graph:]   printable non-space characters
[:cntrl:]   control characters
```

```bash
grep '[[:digit:]]\{1,3\}\.[[:digit:]]\{1,3\}' file  # IP-like pattern (BRE)
grep -E '[[:digit:]]{1,3}\.[[:digit:]]{1,3}' file    # ERE
awk '/[[:upper:]]/'  file    # lines with uppercase letters
```

### PCRE shorthand classes

```
\d    digit [0-9]
\D    non-digit [^0-9]
\w    word char [a-zA-Z0-9_]
\W    non-word char
\s    whitespace [ \t\n\r\f\v]
\S    non-whitespace
\h    horizontal whitespace (space, tab)
\H    non-horizontal whitespace
\v    vertical whitespace (newline, etc.)
\V    non-vertical whitespace
```

```bash
grep -P '\d{4}-\d{2}-\d{2}'  app.log   # date pattern (PCRE)
grep -P '\bERROR\b'           app.log   # whole word ERROR
grep -P '^\s*$'               file      # blank/whitespace lines
```

---

## 4. Quantifiers

### Greedy quantifiers (match as much as possible)

```
*       zero or more
+       one or more  (ERE/PCRE; BRE: \+)
?       zero or one  (ERE/PCRE; BRE: \?)
{n}     exactly n
{n,}    n or more
{n,m}   between n and m (inclusive)
```

```bash
# ERE examples
grep -E 'ab+'     file   # a, then one or more b: ab, abb, abbb
grep -E 'colou?r' file   # color or colour
grep -E 'go{2,4}' file   # goo, gooo, or goooo
grep -E '^.{80,}$' file  # lines 80+ chars long (long lines)
grep -E '^\s{4}'  file   # lines indented 4+ spaces
```

### Lazy (non-greedy) quantifiers (PCRE only)

```
*?      zero or more, as few as possible
+?      one or more, as few as possible
??      zero or one, as few as possible
{n,m}?  between n and m, as few as possible
```

```python
import re
# Greedy: matches everything between first < and last >
re.findall('<.+>',  '<b>bold</b>')   # ['<b>bold</b>']
# Lazy: matches each tag separately
re.findall('<.+?>', '<b>bold</b>')   # ['<b>', '</b>']
```

---

## 5. Groups, Captures & Alternation

### Capturing groups

```
(pattern)           capture group — remembered as \1, \2, etc.
(?:pattern)         non-capturing group — grouping without capture (PCRE/ERE)
(?P<name>pattern)   named capture group (PCRE/RE2/Python)
(?<name>pattern)    named capture group (PCRE alternative syntax)
\1  \2  \3          backreference to captured group
\k<name>            named backreference (PCRE)
```

```bash
# sed backreference — swap date format YYYY-MM-DD → DD/MM/YYYY
echo "2024-01-15" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
# → 15/01/2024

# Extract version from string
echo "app-v1.23.4-linux" | grep -oE '[0-9]+\.[0-9]+\.[0-9]+'
# → 1.23.4

# Wrap a value in quotes
sed -E 's/(password\s*=\s*)(.+)/\1"\2"/' config.conf
```

### Alternation

```
a|b          match a or b (ERE/PCRE; BRE: a\|b)
(cat|dog)    match cat or dog (as a group)
(jpg|jpeg|png|gif|webp)  image extensions
```

```bash
grep -E 'ERROR|FATAL|CRITICAL' app.log
grep -E '\.(jpg|jpeg|png|gif)$' filelist.txt
grep -E '^(GET|POST|PUT|DELETE|PATCH) ' access.log
```

### Non-capturing groups (PCRE/ERE)

```bash
# Group for alternation without capturing
grep -E '(?:ERROR|WARN): .+timeout' app.log
grep -E 'https?://(?:www\.)?example\.com' file

# In sed — group for quantifier, no capture needed
sed -E 's/(?:foo)+/bar/g' file   # replace one or more "foo" with "bar"
# Note: sed -E doesn't support (?:) in all implementations — test on your system
# Use capturing group as workaround:
sed -E 's/(foo)+/bar/g' file
```

---

## 6. Lookahead, Lookbehind & Assertions (PCRE)

### Zero-width assertions

```
(?=pattern)    positive lookahead  — match position followed by pattern
(?!pattern)    negative lookahead  — match position NOT followed by pattern
(?<=pattern)   positive lookbehind — match position preceded by pattern
(?<!pattern)   negative lookbehind — match position NOT preceded by pattern
```

```bash
# grep -P (PCRE) examples

# Lines with "password" NOT followed by " = "  (missing assignment)
grep -P 'password(?!\s*=)' config.conf

# Extract values after "host = " without including "host = "
grep -P '(?<=host = ).+' config.conf

# Port numbers not preceded by ":" (standalone ports)
grep -P '(?<!:)\b[0-9]{4,5}\b' file

# Find "ERROR" only if NOT preceded by "NO "
grep -P '(?<!NO )ERROR' app.log

# Match version numbers only when preceded by "v"
echo "v1.2.3 and 4.5.6" | grep -oP '(?<=v)\d+\.\d+\.\d+'
# → 1.2.3  (not 4.5.6)
```

```python
import re

log = "2024-01-15 ERROR db connection timeout"

# Lookahead: match "ERROR" only if followed by " db"
re.search(r'ERROR(?= db)', log)

# Lookbehind: extract message after "ERROR "
re.search(r'(?<=ERROR ).+', log).group()
# → "db connection timeout"

# Negative lookahead: find IPs not followed by ":80"
re.findall(r'\d{1,3}(?:\.\d{1,3}){3}(?!:80)', text)
```

---

## 7. Flags & Modes

### Common flags

```
i   case-insensitive     grep -i, re.IGNORECASE, /pattern/i
m   multiline            ^ and $ match line boundaries (PCRE/Python)
s   dotall / DOTALL      . matches newline too  (PCRE/Python re.DOTALL)
x   extended / verbose   allow whitespace and comments in pattern
g   global               replace all occurrences (sed s///g, JS /g)
```

```bash
# Case-insensitive grep
grep -i 'error'        app.log
grep -iE 'error|warn'  app.log

# Extended (verbose) regex in Python
import re
pattern = re.compile(r"""
    ^                           # start of line
    (\d{4}-\d{2}-\d{2})         # date: YYYY-MM-DD
    \s+                         # separator
    (\d{2}:\d{2}:\d{2})         # time: HH:MM:SS
    \s+                         # separator
    (ERROR|WARN|INFO|DEBUG)     # log level
    \s+                         # separator
    (.+)$                       # message
""", re.VERBOSE | re.MULTILINE)
```

### Inline flags (PCRE — embed in pattern)

```
(?i)    case insensitive from this point
(?m)    multiline from this point
(?s)    dotall from this point
(?x)    verbose from this point
(?-i)   turn off case insensitive
(?i:pattern)   apply flag to group only
```

```bash
grep -P '(?i)error'         app.log   # case-insensitive inline flag
grep -P '(?i)error(?-i)s'  app.log   # case-insensitive "error", then literal "s"
```

---

## 8. grep & egrep — Log Triage

### Essential flags

```bash
grep -E 'pattern'    file   # extended regex (= egrep)
grep -P 'pattern'    file   # PCRE (Perl-compatible)
grep -F 'string'     file   # fixed string — no regex, fastest
grep -i 'pattern'    file   # case-insensitive
grep -v 'pattern'    file   # invert — lines NOT matching
grep -c 'pattern'    file   # count matching lines
grep -n 'pattern'    file   # show line numbers
grep -l 'pattern'    files  # list filenames with matches
grep -L 'pattern'    files  # list filenames WITHOUT matches
grep -o 'pattern'    file   # print only matched part
grep -h 'pattern'    files  # suppress filename prefix
grep -r 'pattern'    dir/   # recursive
grep -r 'pattern'    dir/ --include='*.log'   # recursive, filter by extension
grep -A 3 'pattern'  file   # 3 lines After match
grep -B 3 'pattern'  file   # 3 lines Before match
grep -C 3 'pattern'  file   # 3 lines Context (before + after)
grep -m 5 'pattern'  file   # stop after 5 matches
grep -q 'pattern'    file   # quiet — exit code only (0=found, 1=not found)
grep --color=always 'pattern' file  # force color (for piping to less -R)
```

### Log triage patterns

```bash
# ── Error hunting ──────────────────────────────────────────────────────────
grep -E 'ERROR|FATAL|CRITICAL|EXCEPTION|panic|PANIC' app.log
grep -iE 'error|fail|fatal|exception|traceback|panic' app.log

# Show context around errors (3 lines before = leads up to error)
grep -B 3 'ERROR' app.log

# Errors in last 1000 lines
tail -1000 app.log | grep -E 'ERROR|FATAL'

# Count errors by type
grep -oE '(ERROR|WARN|INFO|DEBUG)' app.log | sort | uniq -c | sort -rn

# ── Timestamp-based filtering ──────────────────────────────────────────────
# Logs from a specific hour
grep '^2024-01-15 14:' app.log

# Logs between two timestamps (awk is better but grep works for simple cases)
grep -E '^2024-01-15 1[4-6]:' app.log

# ── HTTP access log analysis ────────────────────────────────────────────────
# 5xx errors
grep -E '" 5[0-9]{2} ' access.log

# 4xx errors
grep -E '" 4[0-9]{2} ' access.log

# Specific status codes
grep -E '" (500|502|503|504) ' access.log

# Requests to a specific endpoint
grep '"(GET|POST) /api/v1/users' access.log

# Slow requests (response time > 1s in last field)
awk '$NF > 1.0' access.log | grep -E '" 200 '

# Requests from a specific IP
grep '^10\.0\.0\.1 ' access.log

# ── Security scanning ──────────────────────────────────────────────────────
# Potential SQL injection attempts
grep -iE "('|--|;|union\s+select|or\s+1=1|drop\s+table)" access.log

# Path traversal attempts
grep -E '\.\./|%2e%2e' access.log

# Secrets accidentally committed (in code repos)
grep -rE '(password|passwd|secret|api_key|token)\s*=\s*["\x27][^"\x27]{8,}' \
  --include='*.py' --include='*.js' --include='*.yaml' .

# AWS keys
grep -rE 'AKIA[0-9A-Z]{16}' .

# ── System log patterns ────────────────────────────────────────────────────
# OOM killer
grep -i 'oom\|out of memory\|killed process' /var/log/syslog

# SSH failures
grep 'Failed password\|Invalid user\|authentication failure' /var/log/auth.log

# Disk errors
grep -iE 'i/o error|disk error|bad sector|read error' /var/log/syslog

# ── Pipeline: extract unique IPs from nginx log ────────────────────────────
grep -oE '^[0-9]{1,3}(\.[0-9]{1,3}){3}' access.log | sort -u

# ── Extract and count ──────────────────────────────────────────────────────
# Top 10 IPs by request count
grep -oE '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' access.log \
  | sort | uniq -c | sort -rn | head -10

# Count 5xx errors per minute
grep '" 5[0-9][0-9] ' access.log \
  | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}' \
  | sort | uniq -c
```

### `grep` in scripts

```bash
# Exit code usage — grep returns 0 if match found, 1 if not
if grep -q 'ERROR' app.log; then
  echo "Errors found"
  exit 1
fi

# Silent check (suppress output)
grep -q '^enabled = true' config.conf && echo "enabled"

# Check multiple files, list which have errors
grep -l 'FATAL' /var/log/*.log

# Recursive search with file filter
grep -rn --include='*.conf' 'ssl_certificate' /etc/nginx/
```

---

## 9. sed — Stream Editing & Config Transforms

### Essential syntax

```bash
sed 's/pattern/replacement/'     file   # substitute first occurrence per line
sed 's/pattern/replacement/g'    file   # substitute all occurrences (global)
sed 's/pattern/replacement/i'    file   # case-insensitive (GNU sed)
sed 's/pattern/replacement/2'    file   # substitute 2nd occurrence only
sed 's/pattern/replacement/gi'   file   # global + case-insensitive
sed -n 's/pattern/replacement/p' file   # print only lines where substitution happened
sed -i 's/pattern/replacement/g' file   # edit file in-place
sed -i.bak 's/pattern/replacement/g' file  # in-place with .bak backup
sed -E 's/pattern/replacement/g' file   # extended regex
sed -n '10,20p'                  file   # print lines 10-20
sed '10,20d'                     file   # delete lines 10-20
sed '/pattern/d'                 file   # delete matching lines
sed '/pattern/!d'                file   # delete non-matching lines (keep matches)
sed -n '/start/,/end/p'          file   # print between two patterns
sed '/pattern/a\new line'        file   # append line after match
sed '/pattern/i\new line'        file   # insert line before match
sed '/pattern/c\new line'        file   # replace entire line
```

### Replacement references

```
&       entire matched text
\1 \2   captured group references
\u      uppercase next char  (GNU sed)
\l      lowercase next char  (GNU sed)
\U      uppercase to end or \E
\L      lowercase to end or \E
\E      end \U or \L
```

```bash
# Wrap matched text in quotes
sed -E 's/(password = )(.*)/\1"\2"/' config.conf

# Add prefix to matched content
sed 's/^/  /' file          # indent every line 2 spaces
sed 's/^/# /' file          # comment out every line

# Uppercase the matched word
sed -E 's/\b(error)\b/\U\1/gi' file   # error → ERROR

# Extract — print only the matched part using & 
sed -nE 's/.*version: ([0-9.]+).*/\1/p' Chart.yaml

# Delete trailing whitespace
sed 's/[[:space:]]*$//' file
sed -i 's/[[:space:]]*$//' file    # in-place
```

### DevOps config transforms

```bash
# ── Environment substitution ────────────────────────────────────────────────
# Replace placeholder with env var value
sed "s/\${DB_HOST}/${DB_HOST}/g"   config.template > config.conf
sed "s/__APP_VERSION__/${VERSION}/g" values.yaml

# Multiple substitutions — chain with -e
sed -e "s/APP_ENV/production/g" \
    -e "s/APP_VERSION/${VERSION}/g" \
    -e "s/DB_HOST/${DB_HOST}/g" \
    config.template > config.conf

# ── Nginx config manipulation ────────────────────────────────────────────────
# Enable a disabled config line (remove leading #)
sed -i '/^\s*#\s*worker_processes/s/#\s*//' nginx.conf

# Change worker_processes value
sed -i -E 's/^(worker_processes\s+)[0-9]+;/\1auto;/' nginx.conf

# Update server_name
sed -i -E "s/(server_name\s+)[^;]+;/\1${DOMAIN};/" nginx.conf

# ── YAML/properties file updates ────────────────────────────────────────────
# Update a property value (Java properties file)
sed -i -E "s/^(db\.host\s*=\s*).*/\1${DB_HOST}/" application.properties
sed -i -E "s/^(spring\.datasource\.url\s*=\s*).*/\1${DB_URL}/" application.properties

# Update image tag in a Kubernetes YAML
sed -i -E "s|(image:\s+myapp:)[^\s]+|\1${IMAGE_TAG}|" deployment.yaml

# ── Docker/Containerfile ────────────────────────────────────────────────────
# Update FROM version
sed -i -E 's/^(FROM python:)[0-9]+\.[0-9]+/\1'"${PYTHON_VERSION}"'/' Dockerfile

# ── Remove lines ────────────────────────────────────────────────────────────
sed '/^#/d'              file    # remove comment lines
sed '/^[[:space:]]*$/d'  file    # remove blank lines
sed '/^#/d; /^$/d'       file    # remove comments AND blank lines
sed '/#.*/d'             file    # remove lines with comments anywhere

# ── Multi-line operations ────────────────────────────────────────────────────
# Delete line and the one after it
sed -E '/pattern/{N;d}' file

# Join continuation lines (line ending in \)
sed -E ':a; /\\$/{N; s/\\\n//; ba}' file

# Replace block between two markers
sed -i '/BEGIN GENERATED/,/END GENERATED/c\BEGIN GENERATED\nNEW CONTENT\nEND GENERATED' file
```

---

## 10. awk — Structured Log Parsing

### Core awk patterns

```bash
# Syntax: awk 'pattern { action }' file
# Default action: print (if no action given)
# Default FS: any whitespace

awk '{ print $1 }'          file   # print first field
awk '{ print $1, $3 }'      file   # print fields 1 and 3
awk '{ print NR, $0 }'      file   # print line number + entire line
awk 'NR==5'                 file   # print line 5
awk 'NR>=10 && NR<=20'      file   # print lines 10-20
awk '/pattern/'             file   # print matching lines (like grep)
awk '!/pattern/'            file   # print non-matching lines (like grep -v)
awk '/start/,/end/'         file   # print between patterns
awk 'END { print NR }'      file   # count lines
```

### Field separator

```bash
# -F sets field separator
awk -F: '{ print $1 }' /etc/passwd         # colon-separated
awk -F, '{ print $1, $3 }' data.csv        # CSV (simple, no quoting)
awk -F'\t' '{ print $2 }' data.tsv         # tab-separated
awk -F'[,;]' '{ print $1 }' file           # multiple separators

# OFS = output field separator
awk -F: 'BEGIN{OFS=","} { print $1,$3,$6 }' /etc/passwd
```

### Access log parsing

```bash
# Apache/Nginx combined log format:
# IP - - [timestamp] "METHOD /path HTTP/1.1" status bytes "referer" "agent"

# $1=IP $2=- $3=- $4=[timestamp $5=method $6=path $7=protocol $8=status $9=bytes

# Count requests by status code
awk '{ print $9 }' access.log | sort | uniq -c | sort -rn

# Average response size
awk '{ sum += $10; count++ } END { print sum/count " bytes avg" }' access.log

# Filter 5xx errors
awk '$9 ~ /^5/' access.log

# Top 10 URLs
awk '{ print $7 }' access.log | sort | uniq -c | sort -rn | head 10

# Requests per minute
awk '{ print substr($4,2,17) }' access.log \
  | sort | uniq -c | sort -rn | head 20

# Filter by IP range
awk '$1 ~ /^10\.0\.1\./' access.log

# Requests over 1MB
awk '$10 > 1048576 { print $7, $10 }' access.log

# 95th percentile response time (if response_time is last field)
awk '{ print $NF }' access.log \
  | sort -n \
  | awk 'BEGIN{c=0} {a[c++]=$0} END{print a[int(c*0.95)]}'
```

### System log parsing

```bash
# /var/log/syslog format: Month Day HH:MM:SS host process[pid]: message

# Extract process name and count occurrences
awk '{ match($5, /([^[]+)/, a); print a[1] }' /var/log/syslog \
  | sort | uniq -c | sort -rn

# Filter by time range (14:00 to 15:00)
awk '$3 >= "14:00:00" && $3 < "15:00:00"' /var/log/syslog

# Count kernel messages
awk '$5 ~ /^kernel/' /var/log/syslog | wc -l

# SSH login attempts with IP
awk '/Failed password/ { for(i=1;i<=NF;i++) if($i=="from") print $(i+1) }' \
  /var/log/auth.log | sort | uniq -c | sort -rn
```

### awk for structured data extraction

```bash
# Parse key=value log format
# 2024-01-15 14:23:01 level=ERROR service=api latency=1.23 status=500

awk '{
  for (i=1; i<=NF; i++) {
    if (split($i, kv, "=") == 2) {
      data[kv[1]] = kv[2]
    }
  }
  if (data["level"] == "ERROR") {
    print data["service"], data["latency"], data["status"]
  }
  delete data
}' app.log

# Parse JSON-ish single-line logs (jq is better, but awk works for simple cases)
awk -F'"' '/ERROR/ { print $4 }' app.log    # extract value of first key

# Calculate p50/p95/p99 from a column of numbers
awk '{ a[NR]=$1 }
END {
  n=NR
  asort(a)
  print "p50:", a[int(n*0.50)]
  print "p95:", a[int(n*0.95)]
  print "p99:", a[int(n*0.99)]
  print "max:", a[n]
}' latencies.txt
```

---

## 11. Nginx — Regex in Location & Rewrite

### Location block matching order

```nginx
# Priority order (highest to lowest):
# 1. =    exact match
# 2. ^~   prefix match, stop if found (no regex checked)
# 3. ~    case-sensitive regex
# 4. ~*   case-insensitive regex
# 5.      prefix match (longest wins)

server {
    # 1. Exact match — highest priority
    location = /health {
        return 200 "OK";
    }

    # 2. Prefix, no regex — matches /api/ and stops
    location ^~ /api/ {
        proxy_pass http://backend;
    }

    # 3. Case-sensitive regex
    location ~ ^/v[0-9]+/users/ {
        proxy_pass http://user-service;
    }

    # 4. Case-insensitive regex — match image extensions
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2?)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # 5. Prefix fallback
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Regex captures in location

```nginx
# Capture groups accessible as $1, $2, etc.
location ~ ^/api/v([0-9]+)/(.+)$ {
    proxy_pass http://backend/v$1/$2;
}

# Rewrite with capture
location ~ ^/old-path/(.+)$ {
    return 301 /new-path/$1;
}

# Strip prefix
location ~ ^/static/(.*)$ {
    alias /var/www/static/$1;
}
```

### Rewrite rules

```nginx
# rewrite regex replacement [flag]
# Flags: last, break, redirect (302), permanent (301)

server {
    # Force www
    if ($host !~ ^www\.) {
        rewrite ^ https://www.$host$request_uri permanent;
    }

    # Redirect HTTP to HTTPS
    if ($scheme = http) {
        return 301 https://$host$request_uri;
    }

    # Rewrite /api/v1/* to /api/*
    rewrite ^/api/v1/(.*)$ /api/$1 last;

    # Remove trailing slash (except root)
    rewrite ^(.+)/$ $1 permanent;

    # Versioned assets — strip version hash for cache busting
    # /assets/app-abc123.js → /assets/app.js
    rewrite ^/assets/(.+)-[a-f0-9]{6,}\.(js|css)$ /assets/$1.$2 last;

    # Block user agents matching pattern
    if ($http_user_agent ~* (bot|crawl|spider|scraper)) {
        return 403;
    }

    # Block requests without proper content-type for API
    location /api/ {
        if ($content_type !~ ^application/json) {
            return 415;
        }
    }
}
```

### `map` — regex-based variable assignment

```nginx
# map is more efficient than if blocks for conditional logic
http {
    # Map request URI to backend upstream
    map $uri $backend {
        ~^/api/users    user-service;
        ~^/api/orders   order-service;
        ~^/api/payment  payment-service;
        default         default-backend;
    }

    # Map to boolean (0/1)
    map $request_uri $is_static {
        ~*\.(jpg|png|css|js)$   1;
        default                  0;
    }

    # Map origin to CORS allowed header
    map $http_origin $cors_header {
        ~^https?://(www\.)?example\.com$   $http_origin;
        default                             "";
    }

    server {
        location / {
            proxy_pass http://$backend;
            add_header Access-Control-Allow-Origin $cors_header;
        }
    }
}
```

---

## 12. CI/CD — Branch, Path & Trigger Filters

### GitHub Actions

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'          # release/1.0, release/2.x
      - 'feature/*'           # feature/login (one level)
      - '!hotfix/**'          # exclude hotfix branches
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'   # v1.2.3 semver tags
      - 'v*'                       # any tag starting with v
    paths:
      - 'src/**'
      - '!src/**/*.md'             # exclude markdown files
      - 'Cargo.toml'
      - 'Dockerfile'

  pull_request:
    branches:
      - main
      - 'release/**'
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.github/ISSUE_TEMPLATE/**'

# Note: GitHub Actions uses fnmatch glob patterns, NOT regex
# *   matches any characters except /
# **  matches any characters including /
# ?   matches any single character except /
# [abc] character class
# !pattern  negate (exclude)
```

### GitLab CI — `rules` with regex

```yaml
workflow:
  rules:
    # Run on main, release branches, and semver tags
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH =~ /^release\//'
    - if: '$CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/'
    - when: never

deploy_production:
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/'
      when: manual
    - when: never

run_tests:
  rules:
    # Skip if only docs changed
    - if: '$CI_PIPELINE_SOURCE == "push"'
      changes:
        - 'src/**/*'
        - 'tests/**/*'
        - 'Cargo.toml'
    - when: never

# =~  operator: left side is string, right side is regex (RE2)
# Note: GitLab CI uses RE2 — no lookahead/lookbehind
```

### Jenkins — branch filter regex

```groovy
// Jenkinsfile — branch filter
pipeline {
    agent any
    triggers {
        // Poll SCM only for these branches
        pollSCM('H/5 * * * *')
    }
    stages {
        stage('Deploy') {
            when {
                anyOf {
                    branch 'main'
                    branch pattern: 'release/.*', comparator: 'REGEXP'
                    tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
                }
            }
            steps {
                echo "Deploying ${env.BRANCH_NAME}"
            }
        }
    }
}

// Multibranch pipeline — branch source filter
// In Jenkins UI or jenkins.yaml:
// Branch filter: ^(main|develop|release/.*)$
// Tag filter:    ^v[0-9]+\.[0-9]+\.[0-9]+$
```

### Argo CD — app of apps path filter

```yaml
# applicationset with path regex filter
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  generators:
    - git:
        repoURL: https://github.com/org/gitops-repo.git
        revision: HEAD
        directories:
          - path: 'apps/*/overlays/production'
          - path: 'apps/legacy-*'
            exclude: true
  template:
    metadata:
      name: '{{path.basename}}'
```

---

## 13. Prometheus — Relabeling & Alertmanager Routing

### Prometheus `relabel_configs`

```yaml
# relabel_configs use RE2 regex (no lookahead/lookbehind)
# source_labels: list of labels to concatenate (with separator)
# regex: matched against the concatenated value
# target_label: label to write result to
# replacement: template — $1 $2 reference capture groups

scrape_configs:
  - job_name: 'kubernetes-pods'
    relabel_configs:

      # Keep only pods with annotation prometheus.io/scrape=true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"

      # Drop pods with annotation prometheus.io/scrape=false
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: drop
        regex: "false"

      # Use custom port from annotation
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: '([^:]+)(?::\d+)?;(\d+)'      # capture IP ; capture port
        replacement: '$1:$2'
        target_label: __address__

      # Use custom path from annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: '(.+)'

      # Relabel pod name from metadata
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod

      # Relabel namespace
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace

      # Extract app name from pod name (strip random suffix)
      # pod: myapp-deployment-6d8f9b-xkcd2 → app: myapp-deployment
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        regex: '^(.+)-[a-z0-9]+-[a-z0-9]+$'
        replacement: '$1'
        target_label: app

      # Map pod label to target label
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: replace
        target_label: app

      # Drop metrics with specific label value
      - source_labels: [env]
        action: drop
        regex: 'staging'

      # labelmap — copy all matching labels
      - action: labelmap
        regex: '__meta_kubernetes_pod_label_(.+)'

      # labelkeep — drop all labels NOT matching
      - action: labelkeep
        regex: '(job|instance|pod|namespace|app|env)'

      # labeldrop — drop labels matching
      - action: labeldrop
        regex: '__meta_kubernetes_.*'
```

### Alertmanager — routing regex

```yaml
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h
  receiver: 'default-slack'

  routes:
    # Route by severity — critical pages on-call
    - match_re:
        severity: 'critical|page'
      receiver: 'pagerduty-critical'
      continue: false

    # Route database alerts to db team
    - match_re:
        alertname: '^(PostgreSQL|MySQL|Redis|MongoDB).*'
      receiver: 'db-team-slack'

    # Route by namespace pattern
    - match_re:
        namespace: '^(production|prod-.*)'
        severity: '(warning|critical)'
      receiver: 'prod-alerts-slack'

    # Silence noisy alerts in staging
    - match_re:
        namespace: '^staging.*'
        alertname: '^(CPUThrottling|MemoryHighUsage)$'
      receiver: 'null'    # drop silently

    # Route infrastructure alerts
    - match_re:
        job: '^(node-exporter|kube-.*|etcd)'
      receiver: 'infra-team-slack'

inhibit_rules:
  # Suppress warning if critical is firing for same service
  - source_match_re:
      severity: 'critical'
    target_match_re:
      severity: 'warning'
    equal: ['alertname', 'service', 'namespace']
```

---

## 14. Python — re Module for Automation

### Core functions

```python
import re

# re.search  — find first match anywhere in string → Match or None
# re.match   — match only at start of string → Match or None
# re.fullmatch — match entire string → Match or None
# re.findall — return list of all matches (strings, not Match objects)
# re.finditer — return iterator of Match objects
# re.sub     — replace matches
# re.subn    — replace + return count
# re.split   — split on pattern
# re.compile — compile pattern for reuse

# Always use raw strings r'pattern' to avoid double-escaping
pattern = re.compile(r'(\d{4})-(\d{2})-(\d{2})')

# search
m = re.search(r'ERROR: (.+)', log_line)
if m:
    message = m.group(1)     # first capture group
    full    = m.group(0)     # entire match

# findall — returns list of strings (or tuples if multiple groups)
ips = re.findall(r'\d{1,3}(?:\.\d{1,3}){3}', log_text)

# findall with groups — returns list of tuples
pairs = re.findall(r'(\w+)=(\w+)', 'a=1 b=2 c=3')
# [('a', '1'), ('b', '2'), ('c', '3')]

# Named groups
m = re.search(r'(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})', date_str)
if m:
    year  = m.group('year')
    month = m.group('month')
    d     = m.groupdict()   # {'year': '2024', 'month': '01', 'day': '15'}

# sub — replacement can be string or callable
result = re.sub(r'\bpassword\b', '***', config_text, flags=re.IGNORECASE)

# Callable replacement — access match object
def mask_ip(m):
    parts = m.group().split('.')
    return f"{parts[0]}.{parts[1]}.x.x"

masked = re.sub(r'\d{1,3}(?:\.\d{1,3}){3}', mask_ip, log_text)

# split
parts = re.split(r'[,;\s]+', 'a, b;  c d')
# ['a', 'b', 'c', 'd']
```

### DevOps automation patterns

```python
import re
from pathlib import Path

# ── Parse structured log lines ─────────────────────────────────────────────
LOG_PATTERN = re.compile(r"""
    ^
    (?P<timestamp>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?Z?)
    \s+
    (?P<level>DEBUG|INFO|WARN(?:ING)?|ERROR|FATAL|CRITICAL)
    \s+
    (?P<logger>\S+)
    \s+
    (?P<message>.+?)
    (?:\s+(?P<extra>\{.+\}))?
    $
""", re.VERBOSE)

def parse_log_line(line):
    m = LOG_PATTERN.match(line.strip())
    if m:
        return m.groupdict()
    return None

# ── Extract version from files ──────────────────────────────────────────────
def get_version(filepath):
    text = Path(filepath).read_text()

    patterns = {
        'package.json': r'"version":\s*"([^"]+)"',
        'Cargo.toml':   r'^version\s*=\s*"([^"]+)"',
        'setup.py':     r'version=["\x27]([^"\']+)["\x27]',
        'Chart.yaml':   r'^(?:app)?[Vv]ersion:\s*(.+)',
        'pom.xml':      r'<version>([^<]+)</version>',
    }
    filename = Path(filepath).name
    pattern  = patterns.get(filename, r'version["\s:=]+([0-9]+\.[0-9]+\.[0-9]+)')

    m = re.search(pattern, text, re.MULTILINE)
    return m.group(1).strip() if m else None

# ── Validate config values ───────────────────────────────────────────────────
VALIDATORS = {
    'ip':        re.compile(r'^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$'),
    'cidr':      re.compile(r'^(?:\d{1,3}\.){3}\d{1,3}/(?:[12]?\d|3[0-2])$'),
    'hostname':  re.compile(r'^(?:[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)*[a-zA-Z]{2,}$'),
    'semver':    re.compile(r'^v?(?:0|[1-9]\d*)\.(?:0|[1-9]\d*)\.(?:0|[1-9]\d*)(?:-[a-zA-Z0-9.]+)?(?:\+[a-zA-Z0-9.]+)?$'),
    'docker_tag':re.compile(r'^[a-z0-9]+(?:[._-][a-z0-9]+)*(?::[a-zA-Z0-9._-]+)?$'),
    'k8s_name':  re.compile(r'^[a-z0-9](?:[a-z0-9\-]{0,61}[a-z0-9])?$'),
    'url':       re.compile(r'^https?://[^\s/$.?#].[^\s]*$'),
}

def validate(value, kind):
    return bool(VALIDATORS[kind].match(str(value)))

# ── Scan for secrets in files ────────────────────────────────────────────────
SECRET_PATTERNS = [
    (re.compile(r'(?i)(password|passwd|pwd)\s*[=:]\s*["\x27]?([^\s"\']{8,})'), 'password'),
    (re.compile(r'AKIA[0-9A-Z]{16}'),                                           'aws_access_key'),
    (re.compile(r'(?i)api[_-]?key\s*[=:]\s*["\x27]?([A-Za-z0-9_\-]{20,})'),   'api_key'),
    (re.compile(r'-----BEGIN (?:RSA |EC )?PRIVATE KEY-----'),                   'private_key'),
    (re.compile(r'ghp_[A-Za-z0-9]{36}'),                                        'github_token'),
    (re.compile(r'(?i)secret\s*[=:]\s*["\x27]?([^\s"\']{8,})'),                'secret'),
]

def scan_for_secrets(filepath):
    findings = []
    for i, line in enumerate(Path(filepath).read_text().splitlines(), 1):
        for pattern, kind in SECRET_PATTERNS:
            if pattern.search(line):
                findings.append({'line': i, 'type': kind, 'file': str(filepath)})
    return findings
```

---

## 15. Common DevOps Patterns

### IP and networking

```regex
IPv4 address (simple):
\d{1,3}(?:\.\d{1,3}){3}

IPv4 address (strict 0-255):
(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)

IPv4 CIDR:
(?:\d{1,3}\.){3}\d{1,3}/(?:[12]?\d|3[0-2])

IPv6 (simplified):
(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}
(?:[0-9a-fA-F]{1,4}:)*::(?:[0-9a-fA-F]{1,4}:)*[0-9a-fA-F]{1,4}

Port number (1-65535):
(?:6553[0-5]|655[0-2]\d|65[0-4]\d{2}|6[0-4]\d{3}|[1-5]\d{4}|[1-9]\d{0,3})

Host:port:
[\w.-]+:\d{1,5}

MAC address:
(?:[0-9A-Fa-f]{2}[:-]){5}[0-9A-Fa-f]{2}
```

### Versions and releases

```regex
Semantic version (vX.Y.Z or X.Y.Z):
v?(?:0|[1-9]\d*)\.(?:0|[1-9]\d*)\.(?:0|[1-9]\d*)(?:-[a-zA-Z0-9.]+)?(?:\+[a-zA-Z0-9.]+)?

Simple version (X.Y or X.Y.Z):
[0-9]+\.[0-9]+(?:\.[0-9]+)?

Docker image tag:
[a-z0-9]+(?:[._-][a-z0-9]+)*(?::[a-zA-Z0-9._-]+)?

Git commit SHA (short or long):
[0-9a-f]{7,40}

Git tag (v-prefixed semver):
v[0-9]+\.[0-9]+\.[0-9]+(?:-[a-zA-Z0-9.]+)?
```

### Kubernetes and container

```regex
K8s resource name (RFC 1123):
[a-z0-9](?:[a-z0-9\-]{0,61}[a-z0-9])?

K8s namespace:
[a-z0-9](?:[a-z0-9\-]{0,61}[a-z0-9])?

K8s label value:
[a-zA-Z0-9](?:[a-zA-Z0-9._\-]{0,61}[a-zA-Z0-9])?

Container image (full):
(?:[a-z0-9]+(?:[._-][a-z0-9]+)*(?::[0-9]+)?/)?[a-z0-9]+(?:[._-][a-z0-9]+)*(?:/[a-z0-9]+(?:[._-][a-z0-9]+)*)*(?::[a-zA-Z0-9._-]+)?(?:@sha256:[a-f0-9]{64})?

Docker SHA256 digest:
sha256:[a-f0-9]{64}
```

### Log patterns

```regex
ISO 8601 timestamp:
\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?(?:Z|[+-]\d{2}:\d{2})?

Common log timestamp:
\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}

Log level:
(?:DEBUG|INFO|WARN(?:ING)?|ERROR|FATAL|CRITICAL|TRACE)

Java stack trace start:
^\s+at [\w.$]+\([\w.]+:\d+\)

Exception class:
[A-Z][a-zA-Z0-9]*(?:Exception|Error|Panic|Fault)
```

### Secrets and credentials

```regex
AWS Access Key ID:
AKIA[0-9A-Z]{16}

AWS Secret Key (rough):
[0-9a-zA-Z/+]{40}

GitHub Personal Access Token:
ghp_[A-Za-z0-9]{36}

GitHub Actions Token:
ghs_[A-Za-z0-9]{36}

Generic API key:
(?i)api[_-]?key["\s]*[:=]["\s]*[A-Za-z0-9_\-]{20,}

Private key header:
-----BEGIN (?:RSA |EC |OPENSSH )?PRIVATE KEY-----

JWT token:
eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+
```

### File paths and URLs

```regex
Unix absolute path:
/(?:[^/\s]+/)*[^/\s]*

Windows path:
[A-Za-z]:\\(?:[^\\/:*?"<>|\r\n]+\\)*[^\\/:*?"<>|\r\n]*

URL (simple):
https?://[^\s/$.?#].[^\s]*

URL (strict):
https?://(?:[\w-]+\.)+[\w-]+(?:/[\w.~!$&'()*+,;=:@%-]*)*(?:\?[^\s#]*)?(?:#[^\s]*)?

S3 URI:
s3://[a-z0-9][a-z0-9\-.]{1,61}[a-z0-9](?:/[^\s]*)?

ECR image URI:
\d{12}\.dkr\.ecr\.[a-z0-9-]+\.amazonaws\.com/[^\s:]+(?::[^\s]+)?
```

### Date and time

```regex
ISO date (YYYY-MM-DD):
\d{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])

ISO time (HH:MM:SS):
(?:[01]\d|2[0-3]):[0-5]\d:[0-5]\d

RFC 3339 datetime:
\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?(?:Z|[+-]\d{2}:\d{2})

Duration (Go format: 1h30m20s):
(?:\d+h)?(?:\d+m)?(?:\d+s)?(?:\d+ms)?

Cron expression (5-field):
(?:\*|[0-9,\-*/]+)\s+(?:\*|[0-9,\-*/]+)\s+(?:\*|[0-9,\-*/]+)\s+(?:\*|[0-9,\-*/]+)\s+(?:\*|[0-9,\-*/]+)
```

---

## 16. Quick Reference Cheat Sheet

```
─── ANCHORS ──────────────────────────────────────────────────────────────────
^       start of line/string        $     end of line/string
\b      word boundary               \B    non-word boundary
\A      start of string (PCRE)      \Z    end of string (PCRE)

─── METACHARACTERS ───────────────────────────────────────────────────────────
.       any char (except \n)        \     escape next metachar
|       alternation (ERE/PCRE)      \|    alternation (BRE)

─── CHARACTER CLASSES ────────────────────────────────────────────────────────
[abc]   one of a, b, c              [^abc]  not a, b, c
[a-z]   range                       .       any (not newline)
\d      digit [0-9]          (PCRE) \D      non-digit
\w      word [a-zA-Z0-9_]    (PCRE) \W      non-word
\s      whitespace           (PCRE) \S      non-whitespace
[:alpha:] [:digit:] [:alnum:] [:space:] [:upper:] [:lower:]  (POSIX)

─── QUANTIFIERS ──────────────────────────────────────────────────────────────
*       0 or more (greedy)           *?   0 or more (lazy, PCRE)
+       1 or more (ERE/PCRE)         +?   1 or more (lazy, PCRE)
?       0 or 1   (ERE/PCRE)          ??   0 or 1 (lazy, PCRE)
{n}     exactly n                    {n,}  n or more
{n,m}   n to m                       {n,m}? n to m (lazy, PCRE)
BRE: \+ \? \{n,m\}

─── GROUPS ───────────────────────────────────────────────────────────────────
(pat)           capturing group             \1 \2  backreference
(?:pat)         non-capturing (PCRE/ERE)
(?P<name>pat)   named group (PCRE/RE2)      (?P=name)  named backref
(?i:pat)        inline flag for group only

─── LOOKAROUND (PCRE only — not RE2) ─────────────────────────────────────────
(?=pat)   positive lookahead   — followed by pat
(?!pat)   negative lookahead   — NOT followed by pat
(?<=pat)  positive lookbehind  — preceded by pat
(?<!pat)  negative lookbehind  — NOT preceded by pat

─── FLAGS ────────────────────────────────────────────────────────────────────
i   case-insensitive    m   multiline (^ $ match line ends)
s   dotall (. = \n too) x   verbose (allow whitespace/comments)
g   global (all matches, sed/JS)

─── TOOL QUICK MAP ───────────────────────────────────────────────────────────
grep        BRE default; -E for ERE; -P for PCRE; -F for fixed string
sed         BRE default; -E for ERE
awk         ERE (built-in); use gsub/sub/match/gensub
nginx       PCRE (~ and ~* blocks, rewrite, map)
Python re   PCRE-like
Go regexp   RE2 (no lookahead, no backreferences)
Prometheus  RE2
GitLab CI   RE2
```

### Tool flags at a glance

```bash
# grep
grep -E   # ERE
grep -P   # PCRE (GNU grep)
grep -F   # fixed string (no regex — fastest)
grep -i   # case-insensitive
grep -v   # invert match
grep -o   # print only matched part
grep -n   # show line numbers
grep -c   # count matches
grep -l   # list matching files
grep -r   # recursive
grep -A/-B/-C N  # context lines after/before/both

# sed
sed 's/pat/rep/g'      # substitute all
sed 's/pat/rep/i'      # case-insensitive (GNU)
sed -i                 # in-place edit
sed -i.bak             # in-place with backup
sed -E                 # ERE
sed -n '/pat/p'        # print only matching lines
sed '/pat/d'           # delete matching lines
# Replacement: & = match, \1\2 = groups, \u\U\l\L = case

# awk
awk -F':'              # field separator
awk 'NR==5'            # line 5
awk '$3 > 100'         # field condition
awk '/pat/{action}'    # pattern + action
awk 'BEGIN{} {} END{}' # lifecycle blocks
# Variables: NR=line#, NF=field count, $0=line, $1..$NF=fields, FS OFS RS ORS
```

### Key rules at a glance

| Rule | Detail |
|------|--------|
| BRE vs ERE | `grep` is BRE by default — `(`, `+`, `?`, `{` need `\`; use `-E` for ERE |
| RE2 has no lookaround | Go, Prometheus, GitLab CI, Kubernetes use RE2 — no `(?=)` or `(?<=)` |
| `-F` for fixed strings | When pattern has no regex metacharacters, `-F` is faster and safer |
| Raw strings in Python | Always use `r'pattern'` — avoids double-escaping `\\d` → `\d` |
| Greedy vs lazy | Default is greedy; append `?` to quantifier for lazy (PCRE only) |
| `grep -o` for extraction | Prints only the matched text, not the whole line — essential for pipelines |
| `sed -i.bak` in scripts | Always make a backup before in-place edits in automation |
| Anchor your patterns | `^pattern$` is far more precise than `pattern` — avoids partial matches |
| Test before deploying | Use `grep --color`, Python `re` interactively, or regex101.com |
| POSIX classes in awk | `[:digit:]` etc. are portable across platforms; `\d` is PCRE only |

---

*Part of DevOpsNotes / LANGUAGES — see also `06_TOML_XML.md`, `08_Makefile.md`, `09_QueryLanguages.md`*