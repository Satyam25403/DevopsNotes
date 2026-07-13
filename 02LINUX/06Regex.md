# Regular Expressions (Regex)

Comprehensive guide to regular expressions for pattern matching in Linux text processing tools.

## Table of Contents
- [Basic Regex Symbols](#basic-regex-symbols)
- [Character Classes](#character-classes)
- [Quantifiers](#quantifiers)
- [Extended Regex (ERE)](#extended-regex-ere)
- [Anchors and Boundaries](#anchors-and-boundaries)
- [Grouping and Capturing](#grouping-and-capturing)
- [Practical Examples](#practical-examples)
- [Tool-Specific Usage](#tool-specific-usage)

---

## Basic Regex Symbols

| Symbol | Meaning | Example | Matches |
|--------|---------|---------|---------|
| `.` | Any single character | `a.c` | abc, a1c, a@c |
| `*` | 0 or more of previous | `ab*c` | ac, abc, abbc |
| `^` | Start of line | `^ERROR` | Lines starting with ERROR |
| `$` | End of line | `;$` | Lines ending with ; |
| `\` | Escape character | `\.` | Literal dot (.) |

### Detailed Explanations

#### Dot (.)
**Placeholder for one character**

```bash
# Match any 3-letter word starting with 'c' and ending with 't'
grep 'c.t' file.txt
# Matches: cat, cot, cut, c1t, c@t

# Match filenames
ls | grep 'file.txt'
# Matches: file1txt, file.txt, fileAtxt

# To match literal dot, escape it
grep 'version\.' file.txt
# Matches: version.
```

#### Asterisk (*)
**0 or more occurrences of the previous item**

```bash
# Match lines with 0 or more spaces before ERROR
grep ' *ERROR' log.txt
# Matches: ERROR, ERROR,  ERROR,   ERROR

# Match any number of digits
grep '[0-9]*' file.txt

# Match repeated characters
grep 'o*ps' file.txt
# Matches: ps, ops, oops, ooops
```

⚠️ **Common Mistake:**
```bash
# WRONG: * alone doesn't mean "anything"
grep '*' file.txt        # ERROR: nothing to repeat

# CORRECT: use .* for "anything"
grep '.*' file.txt       # Matches any line
```

#### Caret (^)
**Start of line**

```bash
# Lines starting with ERROR
grep '^ERROR' app.log

# Lines starting with # (comments)
grep '^#' script.sh

# Lines starting with digit
grep '^[0-9]' data.txt
```

#### Dollar ($)
**End of line**

```bash
# Lines ending with semicolon
grep ';$' code.js

# Lines ending with .log
grep '\.log$' files.txt

# Empty lines (start = end)
grep '^$' file.txt
```

#### Backslash (\)
**Escape special characters**

```bash
# Match literal dot
grep '\.' file.txt

# Match literal asterisk
grep '\*' file.txt

# Match dollar sign
grep '\$' prices.txt

# Match backslash
grep '\\' paths.txt
```

---

## Character Classes

### Bracket Expressions []

**Match any ONE character from the set**

```bash
# Single digit
[0-9]           # Matches: 0, 1, 2, ..., 9

# Single letter (lowercase)
[a-z]           # Matches: a, b, c, ..., z

# Single letter (uppercase)
[A-Z]           # Matches: A, B, C, ..., Z

# Any letter
[a-zA-Z]        # Matches: a-z or A-Z

# Specific characters
[aeiou]         # Matches any vowel
[0-9a-f]        # Matches hex digit
```

### Negated Character Classes [^]

**Match any character EXCEPT those in the set**

```bash
# Non-alphabetic character
[^a-zA-Z]       # Matches: numbers, symbols, spaces

# Non-digit
[^0-9]          # Matches: anything except 0-9

# Not a vowel
[^aeiou]        # Matches: consonants and non-letters
```

### Examples

```bash
# Match IP-like patterns
grep '[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*' file.txt

# Match email-like patterns
grep '[a-zA-Z0-9]*@[a-zA-Z]*\.[a-z]*' contacts.txt

# Lines with non-alphabetic characters
grep '[^a-zA-Z]' file.txt

# Find lines with special characters
grep '[^a-zA-Z0-9 ]' data.txt
```

---

## Quantifiers

### Basic Quantifiers

```bash
*           # 0 or more
\+          # 1 or more (needs escape in basic regex)
\?          # 0 or 1 (optional) (needs escape)
```

### Exact Count Quantifiers

| Pattern | Meaning | Example | Matches |
|---------|---------|---------|---------|
| `\{n\}` | Exactly n times | `[0-9]\{3\}` | 123, 456, 789 |
| `\{n,\}` | n or more times | `a\{2,\}` | aa, aaa, aaaa |
| `\{n,m\}` | Between n and m | `[0-9]\{2,4\}` | 12, 123, 1234 |

### Examples

```bash
# Exactly 3 digits (like area codes)
grep '[0-9]\{3\}' phone.txt

# At least 2 consecutive 'o'
grep 'o\{2,\}' words.txt
# Matches: book, foot, ooze

# Between 2 and 4 digits
grep '[0-9]\{2,4\}' data.txt
# Matches: 12, 123, 1234 (but not 1 or 12345)

# ZIP codes (exactly 5 digits)
grep '[0-9]\{5\}' addresses.txt

# Phone numbers (3 digits, dash, 4 digits)
grep '[0-9]\{3\}-[0-9]\{4\}' contacts.txt
```

---

## Extended Regex (ERE)

Extended regex mode (`-E` flag) enables additional features without escaping.

### Enabled with -E

| Feature | Basic (BRE) | Extended (ERE) |
|---------|-------------|----------------|
| 1 or more | `\+` | `+` |
| 0 or 1 | `\?` | `?` |
| OR | `\|` | `|` |
| Grouping | `\(\)` | `()` |
| Exact count | `\{n\}` | `{n}` |

### Alternation (OR)

**Match one pattern OR another**

```bash
# Using -E flag
grep -E 'error|failed|timeout' app.log

# Without -E (needs escaping)
grep 'error\|failed\|timeout' app.log

# Multiple alternatives
grep -E 'ERROR|WARN|FATAL' app.log

# Complex patterns
grep -E '^(http|https|ftp)://' urls.txt
```

### Plus (+)

**Match at least one occurrence**

```bash
# One or more digits (with -E)
grep -E '[0-9]+' file.txt
# Matches: 1, 12, 987 (but not empty)

# Without -E
grep '[0-9]\+' file.txt

# One or more spaces
grep -E ' +' file.txt

# One or more word characters
grep -E '[a-zA-Z]+' file.txt
```

### Question Mark (?)

**Zero or one time (optional)**

```bash
# Optional 's' (http or https)
grep -E 'https?' urls.txt
# Matches: http, https

# Optional area code
grep -E '\(?[0-9]{3}\)?' phones.txt
# Matches: 123, (123)

# Color/Colour
grep -E 'colou?r' text.txt
# Matches: color, colour
```

---

## Anchors and Boundaries

### Line Anchors

```bash
^           # Start of line
$           # End of line

# Lines starting with ERROR
grep '^ERROR' app.log

# Lines ending with .txt
grep '\.txt$' files.txt

# Exact match (entire line)
grep '^exact match$' file.txt

# Empty lines
grep '^$' file.txt

# Non-empty lines
grep -v '^$' file.txt
```

### Word Boundaries (with -E)

```bash
\b          # Word boundary

# Match whole word "error" (not "terror" or "errors")
grep -E '\berror\b' app.log

# Words starting with "test"
grep -E '\btest' file.txt
# Matches: test, testing, tester

# Words ending with "ing"
grep -E 'ing\b' file.txt
# Matches: running, testing
```

---

## Grouping and Capturing

### Parentheses ()

**Group parts of regex together and capture for reuse**

```bash
# Simple grouping (with -E)
grep -E '(ab)+' file.txt
# Matches: ab, abab, ababab

# Capturing for sed replacement
sed -E 's/(DB_HOST)=.*/\1=localhost/' .env
# (DB_HOST) → capture group
# \1 → reuse captured text
# Result: DB_HOST=localhost

# Swap fields
echo "John Doe" | sed -E 's/([A-Z][a-z]+) ([A-Z][a-z]+)/\2, \1/'
# Output: Doe, John

# Extract and reformat
sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/' dates.txt
# 2024-01-15 → 15/01/2024
```

### Multiple Capture Groups

```bash
# Extract multiple parts
echo "User: john, ID: 1234" | sed -E 's/User: ([a-z]+), ID: ([0-9]+)/Name=\1 UserID=\2/'
# Output: Name=john UserID=1234

# Reorder fields
echo "2024-01-15 10:30:45" | sed -E 's/([0-9-]+) ([0-9:]+)/Time: \2, Date: \1/'
# Output: Time: 10:30:45, Date: 2024-01-15
```

---

## Practical Examples

### Log Analysis

```bash
# Find errors or warnings
grep -E '^(ERROR|WARN)' app.log

# Match timestamps
grep -E '[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}' app.log

# Extract IP addresses
grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' access.log

# HTTP status codes (4xx or 5xx)
grep -E 'HTTP/[0-9.]+ [45][0-9]{2}' access.log
```

### Data Validation

```bash
# Email pattern (simple)
grep -E '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$' emails.txt

# Phone number (US format)
grep -E '^\(?[0-9]{3}\)?[-. ]?[0-9]{3}[-. ]?[0-9]{4}$' phones.txt

# ZIP code (5 or 9 digits)
grep -E '^[0-9]{5}(-[0-9]{4})?$' zipcodes.txt

# Credit card (4 groups of 4 digits)
grep -E '^[0-9]{4}[ -]?[0-9]{4}[ -]?[0-9]{4}[ -]?[0-9]{4}$' cards.txt
```

### File Processing

```bash
# Lines with URLs
grep -E 'https?://[a-zA-Z0-9./?=_-]+' file.txt

# Lines with IPv4 addresses
grep -E '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' network.log

# Extract email addresses
grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' contacts.txt
# -o: only matching part

# Find duplicate words
grep -E '\b([a-zA-Z]+) \1\b' document.txt
```

### Configuration Files

```bash
# Find variable assignments
grep -E '^[A-Z_]+=' config.env

# Match commented lines
grep -E '^[ \t]*#' config.txt

# Find key=value pairs
grep -E '^[a-zA-Z_][a-zA-Z0-9_]*=.+$' settings.conf

# Extract values
sed -nE 's/^DB_HOST=(.+)/\1/p' .env
```

---

## Tool-Specific Usage

### grep with Regex

```bash
# Basic regex (default)
grep 'pattern' file.txt

# Extended regex
grep -E 'pattern' file.txt

# Perl-compatible regex (more features)
grep -P 'pattern' file.txt

# Case-insensitive
grep -i 'pattern' file.txt

# Show only matching part
grep -o 'pattern' file.txt
```

### sed with Regex

```bash
# Basic regex (default)
sed 's/old/new/' file.txt

# Extended regex
sed -E 's/pattern/replacement/' file.txt

# Back-references
sed -E 's/(pattern)/\1/' file.txt

# Multiple patterns
sed -E 's/pattern1/repl1/; s/pattern2/repl2/' file.txt
```

### awk with Regex

```bash
# Pattern matching
awk '/regex/ {print}' file.txt

# Field matching
awk '$1 ~ /regex/ {print}' file.txt

# Negation
awk '$1 !~ /regex/ {print}' file.txt

# Case-insensitive
awk 'tolower($0) ~ /pattern/ {print}' file.txt
```

---

## Common Patterns Library

### Network

```bash
# IPv4 address
[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}

# MAC address
([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}

# URL
https?://[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}(/[a-zA-Z0-9._-]*)*

# Domain
[a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.[a-zA-Z]{2,}
```

### Date and Time

```bash
# ISO date (YYYY-MM-DD)
[0-9]{4}-[0-9]{2}-[0-9]{2}

# Time (HH:MM:SS)
[0-9]{2}:[0-9]{2}:[0-9]{2}

# Timestamp
[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}
```

### Identifiers

```bash
# UUID
[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}

# MD5 hash
[a-f0-9]{32}

# SHA-256
[a-f0-9]{64}
```

### Text

```bash
# Email (simple)
[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}

# Username (alphanumeric + underscore)
[a-zA-Z0-9_]{3,20}

# Hexadecimal color
#[0-9A-Fa-f]{6}
```

---

## Confused Notations Summary

Quick reference for commonly confused regex patterns:

| Pattern | Meaning | Example Matches |
|---------|---------|-----------------|
| `.` | Any single character | a, 1, @, space |
| `a*` | 0 or more 'a' | "", a, aa, aaa |
| `a+` | 1 or more 'a' (ERE) | a, aa, aaa |
| `a?` | 0 or 1 'a' (ERE) | "", a |
| `.*` | Any characters (0+) | "", abc, 123!@# |
| `.+` | Any characters (1+) | a, abc, 123!@# |
| `[0-9]` | Single digit | 0, 5, 9 |
| `[0-9]*` | 0+ digits | "", 1, 123 |
| `[0-9]+` | 1+ digits | 1, 123, 4567 |
| `[0-9]{3}` | Exactly 3 digits | 123, 456, 789 |

---

## Best Practices

### Testing Regex

```bash
# Test before using in production
echo "test string" | grep -E 'pattern'

# Use grep with color to see matches
grep --color=auto -E 'pattern' file.txt

# Test sed substitutions
sed -n 's/pattern/replacement/p' file.txt

# Count matches
grep -c 'pattern' file.txt
```

### Performance

1. **Be specific:** More specific patterns are faster
   ```bash
   # Slower
   grep '.*error.*' huge-file.log
   
   # Faster
   grep 'ERROR:' huge-file.log
   ```

2. **Anchor when possible:**
   ```bash
   # Slower
   grep 'pattern' file.txt
   
   # Faster (if at start)
   grep '^pattern' file.txt
   ```

3. **Use fixed strings when no regex needed:**
   ```bash
   # Use -F for literal strings (faster)
   grep -F 'literal.string' file.txt
   ```

### Readability

1. **Comment complex patterns:**
   ```bash
   # Match email addresses
   EMAIL_PATTERN='[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
   grep -E "$EMAIL_PATTERN" file.txt
   ```

2. **Build patterns incrementally:**
   ```bash
   # Start simple
   grep '[0-9]' file.txt
   
   # Add specificity
   grep '[0-9]\{3\}' file.txt
   
   # Complete pattern
   grep '[0-9]\{3\}-[0-9]\{4\}' file.txt
   ```

3. **Use extended regex for readability:**
   ```bash
   # Hard to read
   grep 'http\|https\|ftp' urls.txt
   
   # Easier to read
   grep -E 'http|https|ftp' urls.txt
   ```

---

## Quick Reference Table

| Need | Basic Regex | Extended Regex (-E) |
|------|-------------|---------------------|
| Any character | `.` | `.` |
| 0 or more | `*` | `*` |
| 1 or more | `\+` | `+` |
| Optional | `\?` | `?` |
| OR | `\|` | `|` |
| Grouping | `\( \)` | `( )` |
| Exactly n | `\{n\}` | `{n}` |
| n or more | `\{n,\}` | `{n,}` |
| n to m | `\{n,m\}` | `{n,m}` |
| Start of line | `^` | `^` |
| End of line | `$` | `$` |
| Digit | `[0-9]` | `[0-9]` |
| Letter | `[a-zA-Z]` | `[a-zA-Z]` |
| Not digit | `[^0-9]` | `[^0-9]` |

---

Regular expressions are powerful tools for pattern matching. Master these basics, practice with real data, and gradually tackle more complex patterns!