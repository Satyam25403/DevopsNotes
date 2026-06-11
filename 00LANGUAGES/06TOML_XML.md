# TOML & XML — DevOps Reference Notes

> **TOML** (Tom's Obvious, Minimal Language) — a config file format designed to be unambiguous and easy to read. Used by **Cargo** (Rust), **pip/Poetry** (Python packaging), **Hugo** (static sites), **Gitea**, and many modern CLI tools. **XML** (eXtensible Markup Language) — a verbose but schema-validatable markup language. Still dominant in **Maven** (Java builds), **Spring** (bean config), **Ansible** (some output), **Jenkins** (job DSL), **Kubernetes** (rarely), and enterprise tooling. Both formats are widely parsed in DevOps pipelines.

---

## Table of Contents

1. [TOML — Core Concepts](#1-toml--core-concepts)
2. [TOML — Types & Syntax](#2-toml--types--syntax)
3. [TOML — Tables & Arrays](#3-toml--tables--arrays)
4. [TOML — Cargo (Rust)](#4-toml--cargo-rust)
5. [TOML — Python Packaging (pyproject.toml)](#5-toml--python-packaging-pyprojecttoml)
6. [TOML — Other DevOps Uses](#6-toml--other-devops-uses)
7. [XML — Core Concepts](#7-xml--core-concepts)
8. [XML — Syntax & Structure](#8-xml--syntax--structure)
9. [XML — Namespaces & Schemas](#9-xml--namespaces--schemas)
10. [XML — Maven (pom.xml)](#10-xml--maven-pomxml)
11. [XML — Spring Bean Configuration](#11-xml--spring-bean-configuration)
12. [XML — Jenkins Job DSL](#12-xml--jenkins-job-dsl)
13. [XML — Querying with XPath & xmllint](#13-xml--querying-with-xpath--xmllint)
14. [Parsing & Transforming in CI/CD Pipelines](#14-parsing--transforming-in-cicd-pipelines)
15. [Common Gotchas & Best Practices](#15-common-gotchas--best-practices)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. TOML — Core Concepts

```
TOML is used by:
  Cargo.toml         → Rust package manifest and workspace config
  pyproject.toml     → Python project metadata, Poetry, pip, Black, isort
  Cargo.lock         → dependency lock file (auto-generated, TOML format)
  Hugo config.toml   → static site configuration
  Gitea app.ini      → (INI-like, TOML-adjacent)
  Rust tools         → rustfmt.toml, clippy.toml, deny.toml
  taplo              → TOML language server / formatter
```

TOML design goals:

```
✅ Human-readable and writable
✅ Unambiguous — one obvious way to express each construct
✅ Maps directly to a hash table (dict/map)
✅ Strongly typed — strings, integers, floats, booleans, dates, arrays, tables
❌ Not a programming language — no variables, no conditionals, no imports
```

File extension: `.toml`
Encoding: UTF-8 (required)
Comments: `#` only (no block comments)

---

## 2. TOML — Types & Syntax

### Comments

```toml
# This is a comment
name = "myapp"  # inline comment
```

### Strings

```toml
# Basic string — backslash escapes apply
name = "Hello, World!"
escaped = "Tab:\there, newline:\nhere"
unicode = "Caf\u00E9"          # Café

# Multi-line basic string — leading newline stripped
description = """
This is a
multi-line string.
"""

# Literal string — NO escape processing (raw)
path = 'C:\Users\admin\config'
regex = '\d+\.\d+'

# Multi-line literal string
sql = '''
SELECT *
FROM users
WHERE active = true
'''
```

### Integers

```toml
count    = 42
negative = -17
large    = 1_000_000        # underscores for readability
hex      = 0xDEADBEEF
octal    = 0o755
binary   = 0b11010110
```

### Floats

```toml
pi        = 3.14159
negative  = -0.001
exponent  = 6.626e-34
large     = 9.109e+31
inf_pos   = inf
inf_neg   = -inf
nan_val   = nan
```

### Booleans

```toml
enabled  = true
disabled = false
# Note: only lowercase true/false — True/False/TRUE/FALSE are invalid TOML
```

### Dates and times

```toml
# Offset date-time (RFC 3339)
created_at  = 2024-01-15T10:30:00Z
updated_at  = 2024-01-15T10:30:00+05:30

# Local date-time (no timezone)
local_dt    = 2024-01-15T10:30:00

# Local date (date only)
build_date  = 2024-01-15

# Local time (time only)
start_time  = 10:30:00
```

---

## 3. TOML — Tables & Arrays

### Tables (equivalent to dicts/objects)

```toml
# Inline table — single line, no trailing comma
coords = { x = 1, y = 2, z = 3 }
server = { host = "localhost", port = 8080 }

# Table header — everything below belongs to this table
[database]
host     = "db.example.com"
port     = 5432
name     = "myapp"
user     = "appuser"
password = "secret"

[cache]
host = "redis.example.com"
port = 6379
ttl  = 3600

# Nested tables
[server.tls]
enabled  = true
cert     = "/etc/ssl/cert.pem"
key      = "/etc/ssl/key.pem"

# Equivalent to:
# server = { tls = { enabled = true, cert = "...", key = "..." } }
```

### Arrays

```toml
# Inline arrays — can span multiple lines
ports    = [80, 443, 8080]
tags     = ["web", "nginx", "production"]
matrix   = [[1, 2], [3, 4], [5, 6]]

# Mixed-type arrays are NOT allowed in TOML
# bad = [1, "two", 3.0]   ← invalid

# Multi-line array (trailing comma allowed)
hosts = [
  "web01.example.com",
  "web02.example.com",
  "web03.example.com",
]
```

### Array of tables — `[[double brackets]]`

```toml
# Each [[section]] creates a new element in an array

[[servers]]
name = "web01"
ip   = "10.0.0.1"
role = "primary"

[[servers]]
name = "web02"
ip   = "10.0.0.2"
role = "replica"

[[servers]]
name = "db01"
ip   = "10.0.0.10"
role = "database"

# Result: servers = [ {name="web01",...}, {name="web02",...}, {name="db01",...} ]

# Nested array of tables
[[products]]
name  = "Widget A"
price = 9.99

  [[products.variants]]
  color = "red"
  sku   = "WA-RED"

  [[products.variants]]
  color = "blue"
  sku   = "WA-BLU"

[[products]]
name  = "Widget B"
price = 14.99
```

### Table ordering rules

```toml
# Tables must be defined before they are used as a prefix
# Standard tables cannot be defined after array-of-table entries

# CORRECT
[owner]
name = "Alice"

[owner.address]         # sub-table defined after parent
city = "Bengaluru"

# WRONG — cannot redefine a table
[owner]
name = "Alice"
[owner]                 # ← duplicate table header — TOML parse error
email = "alice@example.com"
```

---

## 4. TOML — Cargo (Rust)

### `Cargo.toml` — package manifest

```toml
[package]
name        = "my-service"
version     = "0.1.0"
edition     = "2021"           # Rust edition: 2015, 2018, 2021
authors     = ["Alice <alice@example.com>"]
description = "A production-ready web service"
license     = "MIT OR Apache-2.0"
repository  = "https://github.com/org/my-service"
homepage    = "https://example.com"
readme      = "README.md"
keywords    = ["web", "api", "service"]
categories  = ["web-programming"]
exclude     = ["tests/fixtures/large_*", "benches/"]
include     = ["src/**", "Cargo.toml", "README.md"]

# Minimum supported Rust version
rust-version = "1.75"

# Binary targets (executable)
[[bin]]
name = "my-service"
path = "src/main.rs"

[[bin]]
name = "my-cli"
path = "src/bin/cli.rs"

# Library target
[lib]
name    = "my_service_lib"
path    = "src/lib.rs"
crate-type = ["cdylib", "rlib"]    # cdylib for FFI, rlib for Rust

# Example targets
[[example]]
name = "basic_usage"
path = "examples/basic.rs"

# Benchmark targets
[[bench]]
name    = "throughput"
harness = false            # use criterion instead of built-in harness
```

### Dependencies

```toml
[dependencies]
# Version constraint styles
tokio       = "1"              # ~= 1.0.0, compatible with 1.x
serde       = "1.0"           # compatible with 1.0.x
axum        = "0.7.5"         # exact minor version
reqwest     = ">=0.11, <0.13" # range
rand        = "*"             # any version (avoid in production)

# With features
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
reqwest = { version = "0.11", default-features = false, features = ["json", "rustls-tls"] }

# Optional dependency (for feature flags)
openssl = { version = "0.10", optional = true }

# Path dependency (local crate)
my-utils = { path = "../my-utils" }

# Git dependency
my-lib = { git = "https://github.com/org/my-lib.git", branch = "main" }
my-lib = { git = "https://github.com/org/my-lib.git", tag = "v0.3.0" }
my-lib = { git = "https://github.com/org/my-lib.git", rev = "abc1234" }

# Rename a dependency
async-trait-renamed = { package = "async-trait", version = "0.1" }

[dev-dependencies]
# Only for tests and examples
tokio = { version = "1", features = ["full", "test-util"] }
mockall      = "0.12"
assert_matches = "1.5"
criterion    = { version = "0.5", features = ["html_reports"] }

[build-dependencies]
# Only for build.rs
cc           = "1.0"
prost-build  = "0.12"
tonic-build  = "0.10"
```

### Features

```toml
[features]
default  = ["json", "tls"]        # features enabled by default

# Simple feature flags
json     = ["serde/derive", "serde_json"]
tls      = ["rustls", "tokio-rustls"]
metrics  = ["prometheus"]
tracing  = ["tracing-subscriber"]

# Additive: enabling "full" enables all others
full     = ["json", "tls", "metrics", "tracing"]

# Feature that enables an optional dependency
openssl-tls = ["openssl"]

# Enabling features of dependencies
postgres = ["sqlx/postgres"]
mysql    = ["sqlx/mysql"]
```

### Profiles

```toml
[profile.dev]
opt-level     = 0          # 0 = no optimization (fastest compile)
debug         = true       # include debug symbols
overflow-checks = true

[profile.release]
opt-level     = 3          # 0-3, "s" (size), "z" (min size)
debug         = false
lto           = true       # link-time optimization
codegen-units = 1          # max optimization (slower compile)
panic         = "abort"    # smaller binary, no stack unwinding
strip         = "symbols"  # strip debug symbols from binary

[profile.release-debug]
inherits  = "release"
debug     = true           # release + debug symbols for profiling
strip     = "none"

[profile.bench]
inherits  = "release"
debug     = 1

# Profile per dependency (speed up dev builds)
[profile.dev.package."*"]
opt-level = 2              # compile deps at O2, own code at O0

[profile.dev.package.image]
opt-level = 3              # this specific dep needs full optimization
```

### Workspace

```toml
# Root Cargo.toml (workspace manifest)
[workspace]
members = [
  "crates/api",
  "crates/db",
  "crates/worker",
  "crates/shared",
  "tools/cli",
]
exclude = ["playground"]

resolver = "2"             # Cargo feature resolver version (use 2 for new projects)

# Shared dependencies across workspace (Cargo 1.64+)
[workspace.dependencies]
tokio  = { version = "1", features = ["full"] }
serde  = { version = "1", features = ["derive"] }
axum   = "0.7"
anyhow = "1"

[workspace.package]
version    = "0.1.0"
edition    = "2021"
authors    = ["Platform Team"]
license    = "MIT"

# Member crate Cargo.toml references workspace
# crates/api/Cargo.toml:
# [package]
# name = "api"
# version.workspace    = true
# edition.workspace    = true
#
# [dependencies]
# tokio.workspace  = true        ← inherits version + features
# serde.workspace  = true
# axum = { workspace = true, features = ["macros"] }  ← add features
```

### Cargo config (`.cargo/config.toml`)

```toml
# .cargo/config.toml — project-level Cargo settings

[build]
target     = "x86_64-unknown-linux-musl"   # default build target
jobs       = 8                              # parallel jobs
rustflags  = ["-C", "link-arg=-fuse-ld=lld"]  # linker flags

[target.x86_64-unknown-linux-musl]
linker     = "x86_64-linux-musl-gcc"
rustflags  = ["-C", "target-feature=+crt-static"]

[target.aarch64-apple-darwin]
linker     = "clang"

[env]
RUST_LOG   = "info"
DATABASE_URL = "postgres://localhost/dev"

[net]
git-fetch-with-cli = true         # use system git (useful behind proxies)
retry              = 3

[registry]
default = "my-registry"

[registries.my-registry]
index = "https://my-registry.example.com/git/index"

[source.crates-io]
replace-with = "vendored-sources"  # use local vendor/ directory

[source.vendored-sources]
directory = "vendor"

[alias]
b   = "build"
t   = "test"
c   = "check"
r   = "run"
fmt = "fmt --all"
cov = "tarpaulin --out Html"
```

---

## 5. TOML — Python Packaging (pyproject.toml)

### PEP 517/518 build system + metadata

```toml
[build-system]
requires      = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

# Or with Poetry:
# requires      = ["poetry-core"]
# build-backend = "poetry.core.masonry.api"

# Or with Hatch:
# requires      = ["hatchling"]
# build-backend = "hatchling.build"

[project]
name            = "my-package"
version         = "1.2.3"
description     = "A short description"
readme          = "README.md"
license         = { file = "LICENSE" }
authors         = [
  { name = "Alice", email = "alice@example.com" },
]
requires-python = ">=3.10"
keywords        = ["devops", "automation"]
classifiers     = [
  "Development Status :: 5 - Production/Stable",
  "Programming Language :: Python :: 3",
  "License :: OSI Approved :: MIT License",
]

# Runtime dependencies
dependencies = [
  "requests>=2.28",
  "click>=8.0",
  "pydantic>=2.0",
  "boto3>=1.26",
]

[project.optional-dependencies]
dev  = ["pytest>=7", "black", "ruff", "mypy"]
docs = ["mkdocs", "mkdocs-material"]
aws  = ["boto3", "botocore"]

[project.urls]
Homepage      = "https://example.com"
Repository    = "https://github.com/org/my-package"
Documentation = "https://docs.example.com"
"Bug Tracker" = "https://github.com/org/my-package/issues"

[project.scripts]
my-cli = "my_package.cli:main"        # console script entry point

[project.entry-points."my_package.plugins"]
csv  = "my_package.plugins.csv:CSVPlugin"
json = "my_package.plugins.json:JSONPlugin"
```

### Tool configuration in pyproject.toml

```toml
# ── Black (formatter) ────────────────────────────────────────
[tool.black]
line-length    = 88
target-version = ["py310", "py311", "py312"]
include        = '\.pyi?$'
exclude        = '''
/(
  \.git
  | \.venv
  | build
  | dist
)/
'''

# ── Ruff (linter + formatter) ────────────────────────────────
[tool.ruff]
line-length    = 88
target-version = "py310"
src            = ["src", "tests"]

[tool.ruff.lint]
select  = ["E", "F", "W", "I", "N", "UP", "B", "A"]
ignore  = ["E501", "B008"]
fixable = ["ALL"]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]      # allow assert in tests
"__init__.py" = ["F401"]   # allow unused imports in init files

[tool.ruff.lint.isort]
known-first-party = ["my_package"]

# ── MyPy (type checker) ──────────────────────────────────────
[tool.mypy]
python_version         = "3.10"
strict                 = true
ignore_missing_imports = true
disallow_untyped_defs  = true
warn_return_any        = true

[[tool.mypy.overrides]]
module                 = "boto3.*"
ignore_missing_imports = true

# ── Pytest ───────────────────────────────────────────────────
[tool.pytest.ini_options]
testpaths     = ["tests"]
addopts       = "-v --tb=short --cov=src --cov-report=term-missing"
filterwarnings = ["error", "ignore::DeprecationWarning"]
markers       = [
  "slow: marks slow tests",
  "integration: marks integration tests",
]

# ── Coverage ─────────────────────────────────────────────────
[tool.coverage.run]
source  = ["src"]
omit    = ["*/tests/*", "*/migrations/*"]
branch  = true

[tool.coverage.report]
show_missing = true
fail_under   = 80
exclude_lines = [
  "pragma: no cover",
  "if TYPE_CHECKING:",
  "raise NotImplementedError",
]
```

### Poetry-specific sections

```toml
[tool.poetry]
name        = "my-package"
version     = "1.2.3"
description = "A short description"
authors     = ["Alice <alice@example.com>"]
packages    = [{ include = "my_package", from = "src" }]

[tool.poetry.dependencies]
python   = "^3.10"
requests = "^2.28"
click    = "^8.0"

[tool.poetry.group.dev.dependencies]
pytest    = "^7.0"
black     = "^23.0"
ruff      = "^0.1"
mypy      = "^1.0"

[tool.poetry.group.docs.dependencies]
mkdocs          = "^1.5"
mkdocs-material = "^9.0"

[tool.poetry.scripts]
my-cli = "my_package.cli:main"
```

---

## 6. TOML — Other DevOps Uses

### `rustfmt.toml` — Rust formatter config

```toml
edition           = "2021"
max_width         = 100
use_tabs          = false
tab_spaces        = 4
newline_style     = "Unix"
imports_granularity = "Crate"   # group imports by crate
group_imports     = "StdExternalCrate"
wrap_comments     = true
format_code_in_doc_comments = true
```

### `deny.toml` — cargo-deny (license + advisory checks)

```toml
[graph]
targets = [
  { triple = "x86_64-unknown-linux-musl" },
  { triple = "aarch64-unknown-linux-musl" },
]

[advisories]
db-path   = "~/.cargo/advisory-db"
db-urls   = ["https://github.com/rustsec/advisory-db"]
vulnerability = "deny"
unmaintained  = "warn"
yanked        = "deny"
ignore        = [
  "RUSTSEC-2020-0071",    # known false positive
]

[licenses]
allow = [
  "MIT",
  "Apache-2.0",
  "Apache-2.0 WITH LLVM-exception",
  "BSD-2-Clause",
  "BSD-3-Clause",
  "ISC",
  "Unicode-DFS-2016",
]
deny  = ["GPL-2.0", "AGPL-3.0"]
copyleft = "warn"

[bans]
multiple-versions = "warn"
deny = [
  { name = "openssl", reason = "use rustls instead" },
]
skip = [
  { name = "windows-sys", reason = "transitive dep with multiple versions" },
]
```

### Hugo `config.toml`

```toml
baseURL       = "https://example.com/"
languageCode  = "en-us"
title         = "My Site"
theme         = "ananke"

[params]
description   = "My awesome site"
author        = "Alice"

[menu]
  [[menu.main]]
  name       = "Home"
  url        = "/"
  weight     = 1

  [[menu.main]]
  name       = "Blog"
  url        = "/blog/"
  weight     = 2

[markup.goldmark.renderer]
unsafe        = true           # allow raw HTML in Markdown

[outputs]
home          = ["HTML", "RSS", "JSON"]
```

---

## 7. XML — Core Concepts

```
XML is used by:
  Maven          → pom.xml (build, deps, plugins, profiles)
  Spring         → applicationContext.xml (bean wiring — legacy)
  Jenkins        → config.xml, job DSL via API
  Ant            → build.xml (legacy Java builds)
  TestNG / JUnit → test result reports (surefire, junit XML format)
  SOAP / WSDL    → web service definitions (enterprise)
  SVG            → vector graphics
  DocBook        → technical documentation
  XSLT           → XML transformation stylesheets
```

XML is:

```
✅ Hierarchical — elements nest inside elements
✅ Schema-validatable — DTD, XSD, RelaxNG
✅ Namespace-aware — avoid element name collisions
✅ Standard — XPath, XSLT, XQuery for querying/transforming
❌ Verbose — lots of boilerplate angle brackets
❌ Not human-friendly for writing by hand
❌ Ambiguous whitespace handling
```

---

## 8. XML — Syntax & Structure

### Basic structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- This is a comment -->
<root>
  <element attribute="value">text content</element>

  <self-closing-element attr="val" />

  <parent>
    <child>text</child>
    <child>more text</child>
  </parent>

  <!-- CDATA — raw text, not parsed as XML -->
  <script><![CDATA[
    if (a < b && c > d) { doSomething(); }
  ]]></script>
</root>
```

### Element types

```xml
<!-- Element with text content -->
<name>Alice</name>

<!-- Element with attributes -->
<server host="db.example.com" port="5432" />

<!-- Element with mixed content (text + children) -->
<description>
  A <em>very</em> important service.
</description>

<!-- Empty element — two equivalent forms -->
<br />
<br></br>
```

### Attribute rules

```xml
<!-- Attribute values must be quoted (single or double) -->
<element attr="double quotes" />
<element attr='single quotes' />

<!-- Special chars in attributes must be escaped -->
<url value="https://example.com?a=1&amp;b=2" />   <!-- & → &amp; -->
<msg value="Say &quot;hello&quot;" />              <!-- " → &quot; -->
<expr value="a &lt; b &gt; c" />                  <!-- < → &lt; > → &gt; -->

<!-- Entity references -->
&amp;   → &
&lt;    → <
&gt;    → >
&quot;  → "
&apos;  → '
&#64;   → @ (decimal numeric)
&#x40;  → @ (hex numeric)
```

### Well-formed vs valid

```
Well-formed (syntax rules):
  - Single root element
  - All elements properly closed
  - Attributes quoted
  - Case-sensitive tag names
  - No overlapping elements

Valid (schema rules):
  - Well-formed PLUS
  - Conforms to a DTD or XSD schema
  - Required attributes present
  - Correct element order and nesting
```

---

## 9. XML — Namespaces & Schemas

### Namespaces

```xml
<!-- Namespace declared with xmlns -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <!-- Elements in default namespace (no prefix) -->
  <groupId>com.example</groupId>

</project>

<!-- Prefixed namespace -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx">

  <!-- Prefixed elements -->
  <context:component-scan base-package="com.example" />
  <tx:annotation-driven />

</beans>
```

### XSD schema reference

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:noNamespaceSchemaLocation="configuration.xsd">
  <!-- validates against local schema file -->
</configuration>
```

---

## 10. XML — Maven (pom.xml)

### Minimal `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>my-service</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>jar</packaging>        <!-- jar | war | pom | ear -->

  <name>My Service</name>
  <description>A production web service</description>

  <properties>
    <java.version>21</java.version>
    <maven.compiler.source>${java.version}</maven.compiler.source>
    <maven.compiler.target>${java.version}</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <spring-boot.version>3.2.0</spring-boot.version>
    <testcontainers.version>1.19.3</testcontainers.version>
  </properties>
</project>
```

### Parent POM (Spring Boot / corporate BOM)

```xml
<!-- Inherit from Spring Boot parent — manages dependency versions -->
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.2.0</version>
  <relativePath />  <!-- lookup parent from repo, not filesystem -->
</parent>

<!-- OR: import a BOM without inheritance (use when you already have a parent) -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.2.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>testcontainers-bom</artifactId>
      <version>${testcontainers.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### Dependencies

```xml
<dependencies>

  <!-- Compile scope (default) — included in runtime classpath -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- version inherited from parent BOM -->
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>

  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>    <!-- only needed at runtime, not compile -->
  </dependency>

  <!-- Provided scope — available at compile, not bundled (provided by container) -->
  <dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <version>6.0.0</version>
    <scope>provided</scope>
  </dependency>

  <!-- Test scope — only for tests -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>

  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>   <!-- version from BOM -->
  </dependency>

  <!-- Exclude a transitive dependency -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
    <exclusions>
      <exclusion>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
      </exclusion>
    </exclusions>
  </dependency>

  <!-- Optional dependency (not transitively inherited) -->
  <dependency>
    <groupId>com.fasterxml.jackson.module</groupId>
    <artifactId>jackson-module-kotlin</artifactId>
    <optional>true</optional>
  </dependency>

</dependencies>
```

### Build and plugins

```xml
<build>
  <finalName>${project.artifactId}-${project.version}</finalName>

  <!-- Source directories -->
  <sourceDirectory>src/main/java</sourceDirectory>
  <testSourceDirectory>src/test/java</testSourceDirectory>
  <resources>
    <resource>
      <directory>src/main/resources</directory>
      <filtering>true</filtering>   <!-- enables ${property} substitution -->
    </resource>
  </resources>

  <plugins>

    <!-- Spring Boot plugin — package as executable JAR -->
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <configuration>
        <excludes>
          <exclude>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
          </exclude>
        </excludes>
        <image>
          <name>myorg/${project.artifactId}:${project.version}</name>
          <builder>paketobuildpacks/builder-jammy-base</builder>
        </image>
      </configuration>
    </plugin>

    <!-- Compiler plugin — set Java version -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration>
        <source>21</source>
        <target>21</target>
        <release>21</release>
        <compilerArgs>
          <arg>--enable-preview</arg>
        </compilerArgs>
        <annotationProcessorPaths>
          <path>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
          </path>
          <path>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct-processor</artifactId>
            <version>${mapstruct.version}</version>
          </path>
        </annotationProcessorPaths>
      </configuration>
    </plugin>

    <!-- Surefire — unit test runner -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <configuration>
        <includes>
          <include>**/*Test.java</include>
          <include>**/*Tests.java</include>
          <include>**/*Spec.java</include>
        </includes>
        <excludedGroups>integration,slow</excludedGroups>
        <argLine>--enable-preview</argLine>
      </configuration>
    </plugin>

    <!-- Failsafe — integration test runner (runs in verify phase) -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-failsafe-plugin</artifactId>
      <executions>
        <execution>
          <goals>
            <goal>integration-test</goal>
            <goal>verify</goal>
          </goals>
        </execution>
      </executions>
      <configuration>
        <includes>
          <include>**/*IT.java</include>
          <include>**/*IntegrationTest.java</include>
        </includes>
      </configuration>
    </plugin>

    <!-- JaCoCo — code coverage -->
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>${jacoco.version}</version>
      <executions>
        <execution>
          <id>prepare-agent</id>
          <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
          <id>report</id>
          <phase>verify</phase>
          <goals><goal>report</goal></goals>
        </execution>
        <execution>
          <id>check</id>
          <goals><goal>check</goal></goals>
          <configuration>
            <rules>
              <rule>
                <limits>
                  <limit>
                    <counter>LINE</counter>
                    <value>COVEREDRATIO</value>
                    <minimum>0.80</minimum>   <!-- 80% line coverage required -->
                  </limit>
                </limits>
              </rule>
            </rules>
          </configuration>
        </execution>
      </executions>
    </plugin>

    <!-- Versions plugin — check for updates -->
    <!-- mvn versions:display-dependency-updates -->
    <!-- mvn versions:display-plugin-updates -->
    <plugin>
      <groupId>org.codehaus.mojo</groupId>
      <artifactId>versions-maven-plugin</artifactId>
      <version>2.16.2</version>
    </plugin>

  </plugins>
</build>
```

### Profiles

```xml
<profiles>

  <!-- Development profile (default active) -->
  <profile>
    <id>dev</id>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    <properties>
      <spring.profiles.active>dev</spring.profiles.active>
      <log.level>DEBUG</log.level>
    </properties>
  </profile>

  <!-- Production profile -->
  <profile>
    <id>production</id>
    <activation>
      <property>
        <name>env</name>
        <value>production</value>
      </property>
    </activation>
    <properties>
      <spring.profiles.active>production</spring.profiles.active>
      <log.level>WARN</log.level>
    </properties>
    <build>
      <plugins>
        <plugin>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-maven-plugin</artifactId>
          <executions>
            <execution>
              <goals><goal>build-image</goal></goals>
            </execution>
          </executions>
        </plugin>
      </plugins>
    </build>
  </profile>

  <!-- Profile activated by OS -->
  <profile>
    <id>linux</id>
    <activation>
      <os><name>Linux</name></os>
    </activation>
  </profile>

  <!-- Profile activated by file existence -->
  <profile>
    <id>has-docker</id>
    <activation>
      <file><exists>/usr/bin/docker</exists></file>
    </activation>
  </profile>

</profiles>
```

### Multi-module Maven project

```xml
<!-- Parent pom.xml -->
<project>
  <groupId>com.example</groupId>
  <artifactId>my-platform</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>pom</packaging>      <!-- must be pom for parent -->

  <modules>
    <module>api</module>
    <module>service</module>
    <module>domain</module>
    <module>infrastructure</module>
  </modules>

  <!-- Shared dependency versions -->
  <dependencyManagement> ... </dependencyManagement>
  <!-- Shared plugin config -->
  <build> ... </build>
</project>

<!-- Child module: api/pom.xml -->
<project>
  <parent>
    <groupId>com.example</groupId>
    <artifactId>my-platform</artifactId>
    <version>1.0.0-SNAPSHOT</version>
  </parent>

  <artifactId>api</artifactId>     <!-- groupId and version inherited -->

  <dependencies>
    <dependency>
      <groupId>com.example</groupId>
      <artifactId>domain</artifactId>
      <version>${project.version}</version>   <!-- sibling module -->
    </dependency>
  </dependencies>
</project>
```

### Maven CLI

```bash
# Lifecycle phases (each includes all previous)
mvn validate          # validate project structure
mvn compile           # compile source code
mvn test              # run unit tests
mvn package           # create JAR/WAR
mvn verify            # run integration tests + verify
mvn install           # install to local ~/.m2 repo
mvn deploy            # deploy to remote repo

# Common flags
mvn package -DskipTests               # skip test execution (still compile)
mvn package -Dmaven.test.skip=true    # skip compile + execution
mvn package -P production             # activate profile
mvn package -pl api,service           # only specific modules
mvn package -pl api -am               # api + modules it depends on
mvn package -pl api -amd              # api + modules that depend on it
mvn package -T 4                      # 4 parallel threads
mvn package -T 1C                     # 1 thread per CPU core
mvn dependency:tree                   # show dependency tree
mvn dependency:tree -Dincludes=log4j  # filter tree
mvn help:effective-pom                # show resolved POM
mvn versions:display-dependency-updates
mvn spring-boot:run                   # run Spring Boot app
mvn spring-boot:build-image           # build OCI image with Buildpacks
```

---

## 11. XML — Spring Bean Configuration

### Modern Spring (annotation-driven — XML minimal)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="
         http://www.springframework.org/schema/beans
         https://www.springframework.org/schema/beans/spring-beans.xsd
         http://www.springframework.org/schema/context
         https://www.springframework.org/schema/context/spring-context.xsd
         http://www.springframework.org/schema/tx
         https://www.springframework.org/schema/tx/spring-tx.xsd">

  <!-- Enable annotation-based config (@Component, @Service, etc.) -->
  <context:component-scan base-package="com.example" />

  <!-- Enable @Transactional -->
  <tx:annotation-driven transaction-manager="transactionManager" />

  <!-- Load properties file -->
  <context:property-placeholder location="classpath:application.properties" />

</beans>
```

### Legacy explicit bean wiring

```xml
<!-- DataSource bean -->
<bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource"
      destroy-method="close">
  <property name="jdbcUrl"  value="${db.url}" />
  <property name="username" value="${db.username}" />
  <property name="password" value="${db.password}" />
  <property name="maximumPoolSize" value="10" />
</bean>

<!-- Constructor injection -->
<bean id="userRepository" class="com.example.UserRepositoryImpl">
  <constructor-arg ref="dataSource" />
  <constructor-arg value="users" />      <!-- string value -->
  <constructor-arg value="100" type="int" />
</bean>

<!-- Setter injection -->
<bean id="userService" class="com.example.UserServiceImpl">
  <property name="userRepository" ref="userRepository" />
  <property name="cacheEnabled"   value="true" />
  <property name="maxRetries"     value="3" />
</bean>

<!-- Scope -->
<bean id="requestScopedBean" class="com.example.RequestContext" scope="request" />
<bean id="sessionScopedBean" class="com.example.UserSession"   scope="session" />
<bean id="prototype"         class="com.example.Worker"        scope="prototype" />

<!-- Factory method -->
<bean id="myBean" class="com.example.BeanFactory"
      factory-method="createInstance">
  <constructor-arg value="config-value" />
</bean>

<!-- List / set / map injection -->
<bean id="notificationService" class="com.example.NotificationService">
  <property name="channels">
    <list>
      <ref bean="emailChannel" />
      <ref bean="slackChannel" />
    </list>
  </property>
  <property name="config">
    <map>
      <entry key="timeout" value="30" />
      <entry key="retries" value="3" />
    </map>
  </property>
  <property name="allowedDomains">
    <set>
      <value>example.com</value>
      <value>internal.example.com</value>
    </set>
  </property>
</bean>
```

---

## 12. XML — Jenkins Job DSL

### Jenkins `config.xml` (Freestyle job)

```xml
<?xml version='1.1' encoding='UTF-8'?>
<project>
  <description>Build and test my-service</description>
  <keepDependencies>false</keepDependencies>

  <!-- Source Code Management -->
  <scm class="hudson.plugins.git.GitSCM">
    <userRemoteConfigs>
      <hudson.plugins.git.UserRemoteConfig>
        <url>https://github.com/org/my-service.git</url>
        <credentialsId>github-token</credentialsId>
      </hudson.plugins.git.UserRemoteConfig>
    </userRemoteConfigs>
    <branches>
      <hudson.plugins.git.BranchSpec>
        <name>*/main</name>
      </hudson.plugins.git.BranchSpec>
    </branches>
    <extensions>
      <hudson.plugins.git.extensions.impl.CleanBeforeCheckout />
    </extensions>
  </scm>

  <!-- Triggers -->
  <triggers>
    <hudson.triggers.SCMTrigger>
      <spec>H/5 * * * *</spec>   <!-- poll every 5 min -->
    </hudson.triggers.SCMTrigger>
    <hudson.triggers.TimerTrigger>
      <spec>H 2 * * 1-5</spec>   <!-- weekdays at ~2am -->
    </hudson.triggers.TimerTrigger>
  </triggers>

  <!-- Build steps -->
  <builders>
    <hudson.tasks.Shell>
      <command>
        mvn clean verify -Dmaven.test.failure.ignore=false
      </command>
    </hudson.tasks.Shell>
  </builders>

  <!-- Post-build -->
  <publishers>
    <hudson.tasks.junit.JUnitResultArchiver>
      <testResults>**/target/surefire-reports/*.xml</testResults>
      <keepLongStdio>false</keepLongStdio>
    </hudson.tasks.junit.JUnitResultArchiver>

    <hudson.tasks.ArtifactArchiver>
      <artifacts>target/*.jar</artifacts>
      <onlyIfSuccessful>true</onlyIfSuccessful>
    </hudson.tasks.ArtifactArchiver>

    <hudson.plugins.emailext.ExtendedEmailPublisher>
      <recipientList>team@example.com</recipientList>
      <defaultSubject>Build ${BUILD_STATUS}: ${JOB_NAME} #${BUILD_NUMBER}</defaultSubject>
    </hudson.plugins.emailext.ExtendedEmailPublisher>
  </publishers>
</project>
```

### JUnit / Surefire XML report format

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="com.example.UserServiceTest"
           tests="12" failures="1" errors="0" skipped="2"
           time="3.456">

  <testcase name="shouldCreateUser" classname="com.example.UserServiceTest" time="0.123">
    <!-- no children = passed -->
  </testcase>

  <testcase name="shouldRejectInvalidEmail" classname="com.example.UserServiceTest" time="0.045">
    <failure message="expected:<400> but was:<200>" type="org.opentest4j.AssertionFailedError">
      org.opentest4j.AssertionFailedError: expected:&lt;400&gt; but was:&lt;200&gt;
        at com.example.UserServiceTest.shouldRejectInvalidEmail(UserServiceTest.java:89)
    </failure>
  </testcase>

  <testcase name="shouldHandleConcurrency" classname="com.example.UserServiceTest" time="0.000">
    <skipped message="Not implemented yet" />
  </testcase>

  <system-out><![CDATA[
    INFO UserService - Creating user: alice
  ]]></system-out>
</testsuite>
```

---

## 13. XML — Querying with XPath & xmllint

### XPath syntax

```xpath
/root                          select root element
/root/child                    select direct child
//element                      select anywhere in document
//*                            all elements
//element[@attr]               elements with attribute
//element[@attr='value']       elements where attr equals value
//element[@attr!='value']      attribute not equals
//element[position()=1]        first element (1-based)
//element[1]                   shorthand for first
//element[last()]              last element
//element[last()-1]            second to last
//element[price>10]            elements where child text > 10
//element/text()               text content of element
//element/@attribute           attribute value
count(//element)               count matching elements
string(//element)              string value
normalize-space(//element)     trimmed string value
contains(@attr, 'val')         attribute contains string
starts-with(@attr, 'val')      attribute starts with string
//a | //b                      union — both a and b elements
//parent/child[2]              second child of parent
```

### `xmllint` — validate and query XML

```bash
# Validate well-formedness
xmllint --noout file.xml
echo $?    # 0 = valid, non-zero = errors

# Validate against XSD schema
xmllint --schema schema.xsd --noout file.xml

# Pretty-print / format
xmllint --format file.xml
xmllint --format file.xml -o formatted.xml

# XPath query
xmllint --xpath "//dependency/artifactId/text()" pom.xml
xmllint --xpath "//dependency[scope='test']/artifactId/text()" pom.xml
xmllint --xpath "count(//dependency)" pom.xml
xmllint --xpath "//properties/*/text()" pom.xml

# Multiple XPath queries in one pass
xmllint --xpath "//groupId[1]/text()" pom.xml
xmllint --xpath "//version[1]/text()" pom.xml

# XPath with namespace (requires namespace declaration)
xmllint --xpath "/*[local-name()='project']/*[local-name()='version']/text()" pom.xml

# Strip namespaces first (easier querying)
xmllint --format file.xml | sed 's/ xmlns[^"]*"[^"]*"//g' | xmllint --xpath "//version/text()" -
```

### `xpath` and Python for XML parsing in CI

```bash
# Python one-liner — extract value from pom.xml
python3 -c "
import xml.etree.ElementTree as ET
tree = ET.parse('pom.xml')
ns = {'m': 'http://maven.apache.org/POM/4.0.0'}
version = tree.find('m:version', ns).text
print(version)
"

# Extract Maven version in CI
VERSION=$(python3 -c "
import xml.etree.ElementTree as ET
ns = {'m': 'http://maven.apache.org/POM/4.0.0'}
print(ET.parse('pom.xml').find('m:version', ns).text)
")
echo "Building version: $VERSION"

# mvn helper (easier than xmllint for Maven)
mvn help:evaluate -Dexpression=project.version -q -DforceStdout
```

### `xq` — XPath / XSLT via jq-like interface

```bash
# xq wraps xmllint with simpler syntax (install: pip install yq)
xq '.project.version' pom.xml
xq '.project.dependencies.dependency[] | select(.scope == "test") | .artifactId' pom.xml

# Extract all dependency artifactIds
cat pom.xml | xq -r '.project.dependencies.dependency[].artifactId'
```

---

## 14. Parsing & Transforming in CI/CD Pipelines

### Extract Maven version in GitHub Actions / Jenkins

```bash
# Method 1: mvn evaluate (most reliable)
VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)

# Method 2: xmllint with local-name (namespace-safe)
VERSION=$(xmllint --xpath \
  "//*[local-name()='project']/*[local-name()='version']/text()" \
  pom.xml)

# Method 3: grep + sed (fragile but no deps)
VERSION=$(grep -m1 '<version>' pom.xml | sed 's/.*<version>\(.*\)<\/version>.*/\1/')

# Set Maven version in CI
mvn versions:set -DnewVersion=${CI_COMMIT_TAG} -DgenerateBackupPoms=false
```

### Transform XML with XSLT

```xml
<!-- extract-deps.xsl — extract dependency list as CSV -->
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0"
  xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
  xmlns:m="http://maven.apache.org/POM/4.0.0">

  <xsl:output method="text" />

  <xsl:template match="/">
    <xsl:for-each select="//m:dependency">
      <xsl:value-of select="m:groupId" />
      <xsl:text>:</xsl:text>
      <xsl:value-of select="m:artifactId" />
      <xsl:text>:</xsl:text>
      <xsl:value-of select="m:version" />
      <xsl:text>&#10;</xsl:text>
    </xsl:for-each>
  </xsl:template>

</xsl:stylesheet>
```

```bash
# Apply XSLT transformation
xsltproc extract-deps.xsl pom.xml

# Convert XML to JSON (yq)
cat pom.xml | xq '.'

# Validate Surefire test results in CI
FAILURES=$(xmllint --xpath "sum(//testsuite/@failures)" target/surefire-reports/*.xml)
ERRORS=$(xmllint --xpath "sum(//testsuite/@errors)" target/surefire-reports/*.xml)
if [ "$FAILURES" -gt 0 ] || [ "$ERRORS" -gt 0 ]; then
  echo "Tests failed: $FAILURES failures, $ERRORS errors"
  exit 1
fi
```

### TOML parsing in shell / CI

```bash
# Parse TOML with tomli (Python 3.11+ has tomllib built-in)
python3 -c "
import tomllib
with open('Cargo.toml', 'rb') as f:
    data = tomllib.load(f)
print(data['package']['version'])
"

# Get Cargo version in CI
VERSION=$(python3 -c "
import tomllib
with open('Cargo.toml', 'rb') as f:
    print(tomllib.load(f)['package']['version'])
")

# Using dasel (universal data selector — TOML, JSON, YAML, XML)
dasel select -f Cargo.toml -s '.package.version'
dasel select -f pom.xml    -s '.project.version'
dasel select -f values.yaml -s '.image.tag'

# Using taplo (TOML-specific CLI)
taplo get -f Cargo.toml 'package.version'
taplo format Cargo.toml          # format in-place
taplo format --check Cargo.toml  # check without modifying (CI)
taplo lint Cargo.toml            # validate
```

---

## 15. Common Gotchas & Best Practices

### TOML gotchas

```toml
# WRONG — bare keys can only use [A-Za-z0-9_-]
# invalid-key! = true        ← ! not allowed in bare key

# CORRECT — quote the key
"invalid-key!" = true
"127.0.0.1"    = "localhost"

# WRONG — duplicate keys
name = "Alice"
name = "Bob"     # ← TOML parse error — duplicate key

# WRONG — mixing table styles
[server]
host = "localhost"
[server]         # ← cannot redefine
port = 8080

# CORRECT — all keys under one header
[server]
host = "localhost"
port = 8080

# WRONG — value after dotted key redefines
[fruit]
apple.color = "red"
apple = { texture = "smooth" }   # ← error: apple already defined as implicit table

# CORRECT
[fruit.apple]
color   = "red"
texture = "smooth"

# NOTE: integers cannot have leading zeros (unlike YAML)
# octal_ok = 0o755   ← octal with prefix
# bad      = 0755    ← INVALID in TOML
```

### XML gotchas

```xml
<!-- ALWAYS escape special characters in text content and attributes -->
<url>https://example.com?a=1&amp;b=2</url>   <!-- & must be &amp; -->
<code>if (a &lt; b)</code>                    <!-- < must be &lt; -->

<!-- Attribute value ambiguity — use consistent quoting -->
<el attr="it's a value" />          <!-- single quote inside double: OK -->
<el attr='say "hello"' />           <!-- double quote inside single: OK -->
<el attr="say &quot;hello&quot;" /> <!-- escaped double quote -->

<!-- Self-closing vs empty — functionally identical in XML -->
<br />        <!-- preferred in XHTML -->
<br></br>     <!-- valid XML -->

<!-- Whitespace is significant in text content -->
<name>  Alice  </name>   <!-- includes leading/trailing spaces -->
<name>Alice</name>       <!-- no spaces -->

<!-- XML declaration encoding must match actual file encoding -->
<?xml version="1.0" encoding="UTF-8"?>  <!-- file must actually be UTF-8 -->
```

### Maven best practices

```xml
<!-- Always pin plugin versions — never rely on defaults -->
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>3.2.2</version>   <!-- ← always specify -->
</plugin>

<!-- Use properties for all version numbers -->
<properties>
  <jackson.version>2.16.0</jackson.version>
</properties>
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>${jackson.version}</version>   <!-- ← use property -->
</dependency>

<!-- Use dependencyManagement for BOMs in multi-module projects -->
<!-- Never put version numbers in child pom.xml if managed by parent -->
```

### Cargo best practices

```toml
# Commit Cargo.lock for applications, gitignore for libraries
# Applications: reproducible builds → commit lock file
# Libraries: let consumers resolve deps → add to .gitignore

# Use workspace.dependencies to keep versions consistent
# across all crates in a workspace

# Prefer semver-compatible constraints
tokio = "1"        # ← good: accepts any 1.x
tokio = "=1.35.1"  # ← only if you need a specific patch

# Separate dev-dependencies from regular ones
# dev-dependencies are never compiled into the final binary

# Use features to keep binaries small
reqwest = { version = "0.11", default-features = false, features = ["json"] }
```

---

## 16. Quick Reference Cheat Sheet

```toml
# ─── TOML TYPES ──────────────────────────────────────────────────────────────
string      = "double" or 'literal (no escapes)'
multiline   = """...""" or '''...'''
integer     = 42 or 1_000_000 or 0xFF or 0o755 or 0b1010
float       = 3.14 or 6.626e-34 or inf or nan
bool        = true or false        # only lowercase
datetime    = 2024-01-15T10:30:00Z
date        = 2024-01-15
time        = 10:30:00
array       = [1, 2, 3] or ["a", "b"]   # same type only
inline_table = { key = "val", n = 42 }

# ─── TOML TABLES ─────────────────────────────────────────────────────────────
[table]                # defines a table (dict)
[parent.child]         # nested table
[[array_of_tables]]    # appends to an array of tables

# ─── CARGO.TOML KEY SECTIONS ─────────────────────────────────────────────────
[package]              name, version, edition, rust-version, authors
[dependencies]         crate = "version" | { version, features, optional, path, git }
[dev-dependencies]     test/bench only
[build-dependencies]   build.rs only
[features]             name = ["dep/feature", "other-feature"]
[profile.release]      opt-level, lto, codegen-units, panic, strip
[workspace]            members, exclude, resolver
[workspace.dependencies]  shared versions across workspace

# ─── PYPROJECT.TOML KEY SECTIONS ─────────────────────────────────────────────
[build-system]         requires, build-backend
[project]              name, version, dependencies, requires-python
[project.optional-dependencies]  extras
[project.scripts]      entry_point = "module:function"
[tool.black]           line-length, target-version
[tool.ruff]            select, ignore, line-length
[tool.mypy]            strict, python_version, ignore_missing_imports
[tool.pytest.ini_options]  testpaths, addopts, markers
[tool.coverage.run]    source, branch, omit
```

```xml
<!-- ─── XML STRUCTURE ─────────────────────────────────────────────────── -->
<?xml version="1.0" encoding="UTF-8"?>
<root>
  <element attr="val">text</element>
  <self-closing />
  <!-- comment -->
  <![CDATA[ raw content with <special> & chars ]]>
</root>

<!-- ─── ENTITY REFERENCES ────────────────────────────────────────────── -->
&amp;  → &      &lt;   → <      &gt;   → >
&quot; → "      &apos; → '      &#64;  → @ (decimal)

<!-- ─── MAVEN POM STRUCTURE ──────────────────────────────────────────── -->
<project>
  <parent>           inherits from parent POM / BOM
  <groupId>          com.example
  <artifactId>       my-service
  <version>          1.0.0-SNAPSHOT
  <packaging>        jar | war | pom
  <properties>       variable definitions — ${my.prop}
  <dependencyManagement>  version pinning (no actual deps)
  <dependencies>     actual dependencies with scope
  <build><plugins>   build tool configuration
  <profiles>         environment-specific config
  <modules>          child modules (parent pom only)
</project>

<!-- ─── DEPENDENCY SCOPES ────────────────────────────────────────────── -->
compile   default — compile + runtime + test classpath
runtime   runtime + test only (drivers, implementations)
provided  compile only — provided by container (servlet-api)
test      test compile + runtime only
system    like provided but explicit path (avoid)
import    BOM import in dependencyManagement only
```

### Key tooling commands

```bash
# ─── CARGO ────────────────────────────────────────────────────────────────
cargo build                       # debug build
cargo build --release             # optimized build
cargo test                        # run all tests
cargo test -- --nocapture         # show println! output
cargo check                       # type-check without building
cargo clippy                      # lint
cargo fmt                         # format
cargo fmt --check                 # check format (CI)
cargo doc --open                  # generate + open docs
cargo add tokio --features full   # add dependency
cargo remove serde                # remove dependency
cargo update                      # update Cargo.lock
cargo audit                       # check for vulnerabilities
cargo deny check                  # license + advisory check
cargo tree                        # show dependency tree
cargo tree -d                     # show duplicate deps
cargo publish                     # publish to crates.io

# ─── MAVEN ────────────────────────────────────────────────────────────────
mvn clean package                 # clean + build JAR
mvn clean verify                  # clean + build + all tests
mvn test -Dtest=UserServiceTest   # run specific test class
mvn dependency:tree               # show dep tree
mvn help:effective-pom            # show resolved POM
mvn versions:display-dependency-updates
mvn versions:set -DnewVersion=2.0.0
mvn spring-boot:run               # run Spring Boot locally
mvn spring-boot:build-image       # OCI image via Buildpacks

# ─── XML TOOLS ────────────────────────────────────────────────────────────
xmllint --noout file.xml                          # validate well-formedness
xmllint --schema schema.xsd --noout file.xml      # validate against XSD
xmllint --format file.xml                         # pretty-print
xmllint --xpath "//version/text()" pom.xml        # XPath query
xsltproc transform.xsl input.xml                  # apply XSLT
xq '.project.version' pom.xml                     # XPath via xq
dasel select -f pom.xml -s '.project.version'     # dasel universal selector

# ─── TOML TOOLS ───────────────────────────────────────────────────────────
taplo format Cargo.toml           # format
taplo format --check Cargo.toml   # CI format check
taplo lint Cargo.toml             # validate
taplo get -f Cargo.toml 'package.version'
dasel select -f Cargo.toml -s '.package.version'
```

### Key rules at a glance

| Rule | Detail |
|------|--------|
| TOML: only lowercase `true`/`false` | `True`, `TRUE`, `yes` are invalid — parse error |
| TOML: no duplicate keys | Redefining a key is a parse error, even across table headers |
| TOML: `[[double]]` for arrays | Single `[brackets]` is a table, `[[double]]` appends to array |
| TOML: integers have no leading zeros | `0755` is invalid — use `0o755` for octal |
| XML: always escape `&` as `&amp;` | Bare `&` in text content causes parse failure |
| XML: attributes must be quoted | `attr=value` is invalid — must be `attr="value"` |
| XML: one root element | Multiple roots are not well-formed |
| Maven: pin all plugin versions | Unpinned plugins pick up latest at build time — not reproducible |
| Maven: `dependencyManagement` ≠ `dependencies` | Management only pins versions — deps must be declared separately |
| Maven: parent BOM vs import | Inherit with `<parent>` for full defaults; `scope=import` BOM when you already have a parent |
| Cargo: commit `Cargo.lock` for apps | Ensures reproducible builds in CI; gitignore for libraries |
| Cargo: `workspace.dependencies` | Single version source of truth across all workspace crates |
| Cargo: `default-features = false` | Opt out of heavy default features (e.g., OpenSSL in reqwest) |

---

*Part of DevOpsNotes / LANGUAGES — see also `03_HCL.md`, `05_GoTemplates.md`, `07_Regex.md`*