# Terraform Built-in Functions

Terraform has a rich standard library of functions that can be called anywhere an expression is valid. They are **pure functions** — no side effects, same input always gives the same output. Terraform does not provides custom functions

```
Function categories:
  Numeric          →  math operations
  String           →  text manipulation
  Collection       →  list/set/map operations
  Encoding         →  base64, JSON, YAML, CSV
  Filesystem       →  read local files
  Date & Time      →  timestamps, formatting
  Hash & Crypto    →  MD5, SHA, bcrypt, UUID
  IP Network       →  CIDR calculations
  Type Conversion  →  cast between types
  Terraform-specific  →  sensitive, nonsensitive, toset, etc.
```

---

## Numeric Functions

```hcl
abs(-42)          # → 42       — absolute value, removes negative sign
ceil(1.2)         # → 2        — round UP to nearest integer
floor(1.9)        # → 1        — round DOWN to nearest integer
max(5, 10, 3)     # → 10       — largest value from a set of numbers
min(5, 10, 3)     # → 3        — smallest value from a set of numbers
pow(2, 8)         # → 256      — 2 to the power of 8
log(100, 10)      # → 2        — log base 10 of 100
signum(-5)        # → -1       — returns -1 (negative), 0 (zero), or 1 (positive)
parseint("FF", 16) # → 255     — parse string as integer in given base (hex here)
```

**Spread operator `...`** — when a function accepts multiple arguments but you have a list, use `...` to expand it:

```hcl
max([10, 50, 30]...)   # → 50  — expands list into individual arguments
min([10, 50, 30]...)   # → 10
```

---

## String Functions

```hcl
lower("Hello WORLD")           # → "hello world"       — all lowercase
upper("hello world")           # → "HELLO WORLD"       — all uppercase
trimspace("  hello  ")         # → "hello"             — strip leading/trailing whitespace
trim("##hello##", "#")         # → "hello"             — strip specific chars from both ends
trimprefix("dev-rg", "dev-")   # → "rg"                — remove prefix if present
trimsuffix("rg-dev", "-dev")   # → "rg"                — remove suffix if present

replace("project name!", " ", "-")   # → "project-name!"  — replace all occurrences
replace("project name!", "!", "")    # → "project name"   — delete character

substr("techtutorials", 0, 4)   # → "tech"  — (string, offset, length)
length("hello")                 # → 5       — character count for strings

split(",", "80,443,3306")       # → ["80", "443", "3306"]   — string → list
join("-", ["a", "b", "c"])      # → "a-b-c"                 — list → string

format("%-10s", "dev")          # → "dev       "            — printf-style formatting
format("%05d", 42)              # → "00042"

strcontains("standard_D2s", "standard")   # → true  — substring check
startswith("dev-rg", "dev")               # → true
endswith("test_backup", "_backup")        # → true

indent(2, "line1\nline2")       # → "line1\n  line2"  — indent all lines except first
chomp("hello\n")                # → "hello"            — remove trailing newline
```

---

## Collection Functions

```hcl
length(["a", "b", "c"])         # → 3    — count elements
length({ a = 1, b = 2 })        # → 2    — count keys in map

contains(["dev","prod"], "dev")  # → true   — check if element exists in list
element(["a","b","c"], 1)        # → "b"    — safe index access (wraps around)
index(["a","b","c"], "b")        # → 1      — find position of element

flatten([["a","b"], ["c"]])      # → ["a","b","c"]      — collapse nested lists
distinct(["a","b","a","c"])      # → ["a","b","c"]      — remove duplicates from list
compact(["a","","b","","c"])     # → ["a","b","c"]      — remove empty strings
reverse(["a","b","c"])           # → ["c","b","a"]      — reverse a list
sort(["c","a","b"])              # → ["a","b","c"]      — lexicographic sort
slice(["a","b","c","d"], 1, 3)  # → ["b","c"]          — (list, start_index, end_index exclusive)

concat(["a","b"], ["c","d"])     # → ["a","b","c","d"]  — combine lists
toset(["a","b","a"])             # → {"a","b"}          — list → set (removes duplicates)
tolist(toset(["b","a"]))         # → ["a","b"]          — set → list (sorted)

# Map functions
merge({a=1}, {b=2}, {a=3})      # → {a=3, b=2}    — later maps override earlier on key conflict
keys({a=1, b=2})                # → ["a","b"]     — extract keys as list
values({a=1, b=2})              # → [1, 2]        — extract values as list
lookup({a=1, b=2}, "a", 0)      # → 1             — safe key access with default fallback instead of using if-else
lookup({a=1, b=2}, "c", 0)      # → 0             — key missing, returns default

zipmap(["a","b"], [1, 2])        # → {a=1, b=2}   — create map from keys list + values list
```

---

## Encoding Functions

```hcl
base64encode("hello terraform")        # → "aGVsbG8gdGVycmFmb3Jt"
base64decode("aGVsbG8gdGVycmFmb3Jt")  # → "hello terraform"

jsonencode({ env = "dev", port = 80 })
# → "{\"env\":\"dev\",\"port\":80}"

jsondecode("{\"env\":\"dev\",\"port\":80}")
# → { env = "dev", port = 80 }

yamlencode({ env = "dev", port = 80 })
# → "env: dev\nport: 80\n"

yamldecode("env: dev\nport: 80")
# → { env = "dev", port = 80 }

csvdecode("name,age\nalice,30\nbob,25")
# → [{ name="alice", age="30" }, { name="bob", age="25" }]

urlencode("hello world/path?q=1")
# → "hello+world%2Fpath%3Fq%3D1"
```

---

## Filesystem Functions

```hcl
file("./scripts/init.sh")          # read entire local file into a string
fileexists("./config.json")        # → true/false — check if file exists

filebase64("./certs/cert.pem")     # read file and base64-encode it in one step

# Useful for injecting startup scripts into VMs:
custom_data = base64encode(file("./scripts/cloud-init.sh"))

# Read and decode JSON config file:
local.config_content = file("config.json")
jsondecode(local.config_content)   # parse into a map
```

---

## Date and Time Functions

```hcl
timestamp()    # → "2025-01-15T10:30:00Z"  — current UTC time in RFC 3339 format

# formatdate(format, timestamp) — format timestamp into a custom string
formatdate("YYYYMMDD", timestamp())       # → "20250115"   — compact date for resource names
formatdate("DD-MM-YYYY", timestamp())     # → "15-01-2025"  — human-readable date for tags
formatdate("YYYY-MM-DD hh:mm:ss", timestamp())  # → "2025-01-15 10:30:00"

timeadd(timestamp(), "24h")   # → timestamp + 24 hours  — add duration to a time
timeadd(timestamp(), "-1h")   # → timestamp - 1 hour

timecmp("2025-01-01T00:00:00Z", "2025-06-01T00:00:00Z")  # → -1 (first is earlier)
```

> `timestamp()` is re-evaluated on every `terraform plan` and `terraform apply` — so a resource using it directly will always show as changed. Store it in a `local` to use it consistently within one run.

---

## Hash and Crypto Functions

```hcl
md5("hello")              # → "5d41402abc4b2a76b9719d911017c592"   — MD5 hash (not secure, use for checksums)
sha1("hello")             # → "aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d"
sha256("hello")           # → "2cf24db..."                          — SHA-256, use for integrity checks
sha512("hello")           # → long hex string

bcrypt("mypassword")      # → "$2a$10$..."  — bcrypt hash (for passwords)

uuid()                    # → "b5ee72a3-54dd-4e95-b083-f8a71c4b7e5e"  — random UUID v4
uuidv5("dns","example.com")  # → deterministic UUID from namespace + name
```

---

## IP Network Functions

```hcl
cidrhost("10.0.0.0/16", 10)      # → "10.0.0.10"      — nth host in CIDR block
cidrnetmask("10.0.0.0/16")       # → "255.255.0.0"    — subnet mask
cidrsubnet("10.0.0.0/8", 8, 2)   # → "10.2.0.0/16"    — (prefix, newbits, netnum)

# cidrsubnet explained:
# "10.0.0.0/8"  → base CIDR
# 8             → add 8 bits to prefix (/8 becomes /16)
# 2             → the 2nd subnet at that size → 10.2.0.0/16

cidrsubnets("10.0.0.0/8", 8, 8, 4)
# → ["10.0.0.0/16", "10.1.0.0/16", "10.2.0.0/12"]  — allocate multiple subnets at once

cidrcontains("10.0.0.0/16", "10.0.1.5")  # → true  — check if IP is within CIDR
```

---

## Type Conversion Functions

```hcl
tostring(42)        # → "42"     — number → string
tonumber("42")      # → 42       — string → number
tobool("true")      # → true     — string → bool
tobool("false")     # → false

tolist(toset(["b","a","c"]))   # → ["a","b","c"]  — set → list (alphabetically sorted)
toset(["a","b","a"])           # → {"a","b"}       — list → set (removes duplicates)
tomap({ a = 1, b = 2 })        # → map of numbers  — ensure value is map type

# can() — test if an expression is valid without erroring
can(tonumber("abc"))    # → false  — "abc" is not a valid number, but no error thrown
can(tonumber("42"))     # → true

# try() — return first expression that doesn't error
try(tonumber("abc"), 0)   # → 0       — "abc" fails, falls back to 0
try(tonumber("42"), 0)    # → 42      — succeeds, returns 42
try(var.optional_map.key, "default")  # safe nested access with fallback
```

---

## Terraform-Specific Functions

```hcl
# sensitive() — mark a value as sensitive so Terraform redacts it from output
sensitive("my-secret-password")   # value is hidden in plan/apply output

# nonsensitive() — unwrap a sensitive value (use carefully — exposes the value)
nonsensitive(local.config_content)   # used when you intentionally want to output it

# templatefile(path, vars) — render a template file with variable substitution
templatefile("./templates/cloud-init.tpl", {
  hostname = "web-server-01"
  packages = ["nginx", "curl"]
})

# Example template file (cloud-init.tpl):
# #!/bin/bash
# hostname ${hostname}
# apt-get install -y ${join(" ", packages)}

# plantimestamp() — like timestamp() but only evaluated during plan, not apply
# Useful for plan-time annotations that shouldn't drift on apply
```

---

## Combined Demonstration

All the functions above working together across the full project.

### `variables.tf`

```hcl
variable "project_name" {
  type        = string
  description = "Name of the project"
  default     = "Project ALPHA Resource"
}

variable "default_tags" {
  type = map(string)
  default = {
    company    = "CloudOps"
    managed_by = "terraform"
  }
}

variable "environment_tags" {
  type = map(string)
  default = {
    environment = "production"
    cost_center = "cc-123"
  }
}

# Deliberately messy — will be cleaned up with string functions in locals
variable "storage_account_name" {
  type    = string
  default = "techtutorIALS with!piyushthis should be formatted"
}

variable "allowed_ports" {
  type    = string
  default = "80,443,3306"
}

variable "environment" {
  type        = string
  description = "Environment name"
  default     = "dev"
  # validation block is itself an expression — condition must return bool
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Enter a valid value for env: dev, staging, or prod"
  }
}

variable "vm_sizes" {
  type = map(string)
  default = {
    dev     = "standard_D2s_v3"
    staging = "standard_D4s_v3"
    prod    = "standard_D8s_v3"
  }
}

variable "vm_size" {
  type    = string
  default = "standard_D2s_v3"
  # multiple validation blocks are allowed
  validation {
    condition     = length(var.vm_size) >= 2 && length(var.vm_size) <= 20
    error_message = "vm_size must be between 2 and 20 characters"
  }
  validation {
    condition     = strcontains(lower(var.vm_size), "standard")
    error_message = "vm_size must contain the word 'standard'"
  }
}

variable "backup_name" {
  type    = string
  default = "test_backup"
  validation {
    condition     = endswith(var.backup_name, "_backup")
    error_message = "backup_name must end with '_backup'"
  }
}

variable "credential" {
  type      = string
  default   = "xyz123"
  sensitive = true   # redacted in all plan/apply output
}
```

### `locals.tf`

```hcl
locals {
  # ── String functions ────────────────────────────────────────────────────────

  # lower() + replace() — "Project ALPHA Resource" → "project-alpha-resource"
  # Used as the base name for all resources
  formatted_name = lower(replace(var.project_name, " ", "-"))

  # Multi-step string cleanup pipeline:
  # 1. substr()   — truncate to 23 chars (Azure storage account max is 24)
  # 2. lower()    — storage names must be lowercase
  # 3. replace()  — remove spaces
  # 4. replace()  — remove "!" characters
  # Input:  "techtutorIALS with!piyushthis should be formatted"
  # Output: "techtutorialswithpiyus"
  storage_formatted = replace(
    replace(
      lower(substr(var.storage_account_name, 0, 23)),
      " ", ""
    ),
    "!", ""
  )

  # ── Map functions ───────────────────────────────────────────────────────────

  # merge() — combine two tag maps into one
  # If the same key appears in both, environment_tags wins (it's listed last)
  merge_tags = merge(var.default_tags, var.environment_tags)

  # ── Collection + String functions ───────────────────────────────────────────

  # split() — "80,443,3306" → ["80", "443", "3306"]
  formatted_ports = split(",", var.allowed_ports)

  # for expression — transform each port string into a rule object
  nsg_rules = [for port in local.formatted_ports : {
    name        = "port-${port}"
    port        = port
    description = "Allowed traffic on port: ${port}"
  }]

  # ── lookup() ────────────────────────────────────────────────────────────────

  # lookup(map, key, default) — get VM size for current environment
  # If var.environment doesn't exist in the map, fall back to "dev" size
  vm_size = lookup(var.vm_sizes, var.environment, lower("dev"))

  # ── Set + concat() ──────────────────────────────────────────────────────────

  user_location    = ["eastus", "westus", "eastus"]   # has a duplicate
  default_location = ["centralus"]

  # concat()  — combine the two lists → ["eastus", "westus", "eastus", "centralus"]
  # toset()   — convert to set → removes duplicate "eastus" → {"centralus","eastus","westus"}
  unique_location = toset(concat(local.user_location, local.default_location))

  # ── Numeric functions ────────────────────────────────────────────────────────

  monthly_costs = [-50, 100, 75, 200]

  # for expression + abs() — convert all values to positive
  # → [50, 100, 75, 200]
  positive_cost = [for cost in local.monthly_costs : abs(cost)]

  # max() with spread operator ... — expand list into individual arguments
  # → 200
  max_cost = max(local.positive_cost...)

  # ── Date and time functions ──────────────────────────────────────────────────

  current_time = timestamp()

  # formatdate() — "20250115" — used in auto-generated resource names
  resource_name = formatdate("YYYYMMDD", local.current_time)

  # formatdate() — "15-01-2025" — human-readable format for tags
  tag_date = formatdate("DD-MM-YYYY", local.current_time)

  # ── Filesystem + sensitive() ─────────────────────────────────────────────────

  # file() reads config.json from disk into a string
  # sensitive() wraps the entire value — Terraform will redact it in output
  config_content = sensitive(file("config.json"))
}
```

### `main.tf`

```hcl
resource "azurerm_resource_group" "rg" {
  # local.formatted_name → "project-alpha-resource"
  name     = "${local.formatted_name}-rg"
  location = "westus2"

  # local.merge_tags → merged map of default + environment tags
  tags = local.merge_tags
}

resource "azurerm_storage_account" "example" {
  # local.storage_formatted → cleaned, lowercase, max-23-char name
  name                     = local.storage_formatted
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
  tags                     = local.merge_tags
}

resource "azurerm_network_security_group" "example" {
  name                = "${local.formatted_name}-nsg"   #string concatenation
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  # dynamic block — generates one security_rule block per element in local.nsg_rules (a list of objects)
  dynamic "security_rule" {
    for_each = local.nsg_rules

    content {
      name                       = security_rule.value.name
      priority                   = 100
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = security_rule.value.port
      source_address_prefix      = "*"
      destination_address_prefix = "*"
      description                = security_rule.value.description
    }
  }
}
```

### `outputs.tf`

```hcl
output "rgname" {
  value = azurerm_resource_group.rg.name
  # → "project-alpha-resource-rg"
}

output "storage_name" {
  value = azurerm_storage_account.example.name
  # → "techtutorialswithpiyus"
}

output "nsg_rules" {
  value = local.nsg_rules
  # → list of {name, port, description} objects for each port
}

output "security_name" {
  value = azurerm_network_security_group.example.name
  # → "project-alpha-resource-nsg"
}

output "vm_size" {
  value = local.vm_size
  # → "standard_D2s_v3" (looked up for "dev" environment)
}

output "backup" {
  value = var.backup_name
  # → "test_backup"
}

output "credential" {
  value     = var.credential
  sensitive = true   # redacted in terraform output — must be marked sensitive here too
}

output "unique_location" {
  value = local.unique_location
  # → {"centralus", "eastus", "westus"}  (set — no duplicates)
}

output "max_cost" {
  value = local.max_cost
  # → 200
}

output "positive" {
  value = local.positive_cost
  # → [50, 100, 75, 200]
}

output "resource_tag" {
  value = local.resource_name
  # → "20250115"  (date-stamped resource name)
}

output "config_loaded" {
  # nonsensitive() unwraps the sensitive wrapper so we can output the decoded JSON
  # Use carefully — this will print the config content in plain text
  value = nonsensitive(jsondecode(local.config_content))
}
```

---

## Functions — Complete Reference Table

| Category | Function | What it does |
|----------|----------|-------------|
| **Numeric** | `abs(n)` | Absolute value |
| | `ceil(n)` | Round up |
| | `floor(n)` | Round down |
| | `max(a,b,...)` | Largest value |
| | `min(a,b,...)` | Smallest value |
| | `pow(x,y)` | x to the power y |
| | `log(n,base)` | Logarithm |
| | `signum(n)` | -1, 0, or 1 |
| | `parseint(s,base)` | Parse string as integer |
| **String** | `lower/upper(s)` | Case conversion |
| | `trimspace(s)` | Strip whitespace |
| | `trim/prefix/suffix(s,c)` | Strip characters |
| | `replace(s,old,new)` | Find and replace |
| | `substr(s,offset,len)` | Slice string |
| | `split(sep,s)` | String → list |
| | `join(sep,list)` | List → string |
| | `format(fmt,...)` | Printf-style format |
| | `strcontains/startswith/endswith` | Substring checks |
| | `length(s)` | Character count |
| **Collection** | `length(col)` | Element count |
| | `contains(list,val)` | Check membership |
| | `element(list,idx)` | Safe index access |
| | `index(list,val)` | Find position |
| | `flatten(lists)` | Collapse nested lists |
| | `distinct(list)` | Remove duplicates |
| | `compact(list)` | Remove empty strings |
| | `concat(a,b,...)` | Combine lists |
| | `sort/reverse(list)` | Order list |
| | `slice(list,s,e)` | Sublist |
| | `merge(maps...)` | Combine maps |
| | `keys/values(map)` | Extract keys/values |
| | `lookup(map,key,def)` | Safe map access |
| | `zipmap(keys,vals)` | Build map from lists |
| **Encoding** | `jsonencode/jsondecode` | JSON serialization |
| | `yamlencode/yamldecode` | YAML serialization |
| | `base64encode/decode` | Base64 encoding |
| | `csvdecode(s)` | CSV → list of maps |
| | `urlencode(s)` | URL encoding |
| **Filesystem** | `file(path)` | Read file to string |
| | `fileexists(path)` | Check file exists |
| | `filebase64(path)` | Read + base64 encode |
| **Date & Time** | `timestamp()` | Current UTC time |
| | `formatdate(fmt,ts)` | Format timestamp |
| | `timeadd(ts,dur)` | Add duration |
| | `timecmp(a,b)` | Compare timestamps |
| **Hash & Crypto** | `md5/sha1/sha256/sha512` | Hash functions |
| | `bcrypt(s)` | Bcrypt hash |
| | `uuid/uuidv5` | Generate UUID |
| **IP Network** | `cidrhost(cidr,n)` | Nth host in CIDR |
| | `cidrnetmask(cidr)` | Subnet mask |
| | `cidrsubnet(cidr,n,i)` | Calculate subnet |
| | `cidrsubnets(cidr,...)` | Allocate subnets |
| | `cidrcontains(cidr,ip)` | IP in range check |
| **Type Conversion** | `tostring/tonumber/tobool` | Primitive casts |
| | `tolist/toset/tomap` | Collection casts |
| | `can(expr)` | Test expression validity |
| | `try(a,b,...)` | First non-error value |
| **Terraform-specific** | `sensitive(val)` | Mark value as sensitive |
| | `nonsensitive(val)` | Unwrap sensitive |
| | `templatefile(path,vars)` | Render template file |