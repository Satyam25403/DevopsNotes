# Jinja2 — DevOps Reference Notes

> **Jinja2** — a fast, expressive, extensible templating engine for Python. In DevOps, Jinja2 is the backbone of **Ansible** (playbooks, roles, `ansible.cfg`), **SaltStack** (states, pillars, grains), and **Cookiecutter** (project scaffolding). It separates static structure from dynamic values, enabling reusable, environment-aware configuration generation.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Delimiter Syntax](#2-delimiter-syntax)
3. [Variables & Filters](#3-variables--filters)
4. [Control Structures](#4-control-structures)
5. [Tests](#5-tests)
6. [Macros & Includes](#6-macros--includes)
7. [Inheritance & Blocks](#7-inheritance--blocks)
8. [Whitespace Control](#8-whitespace-control)
9. [Ansible — Jinja2 in Playbooks](#9-ansible--jinja2-in-playbooks)
10. [Ansible — template Module & Files](#10-ansible--template-module--files)
11. [Ansible — Filters Deep Dive](#11-ansible--filters-deep-dive)
12. [Ansible — Conditionals & Loops](#12-ansible--conditionals--loops)
13. [Ansible — Roles & Variable Precedence](#13-ansible--roles--variable-precedence)
14. [SaltStack — Jinja2 in States](#14-saltstack--jinja2-in-states)
15. [Common Gotchas & Best Practices](#15-common-gotchas--best-practices)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Core Concepts

```
Jinja2 is used by:
  Ansible    → playbooks, templates (.j2), role defaults, handlers
  SaltStack  → state files (.sls), pillars, grains rendering
  Cookiecutter → project scaffolding templates
  Helm       → Go templates (different engine, similar concepts)
  Flask/Django → web app HTML rendering (less relevant for DevOps)
```

Jinja2 templates are plain text files with embedded **tags**:

```jinja2
{# This is a comment — stripped from output #}
{{ variable }}               {# outputs a value #}
{% if condition %}           {# control flow #}
{% for item in list %}       {# iteration #}
```

- File extension: `.j2` (convention for Ansible templates), `.jinja`, `.jinja2`, or just `.conf`/`.yaml` with inline Jinja2
- Jinja2 is **not** a programming language — it's a rendering engine. Logic should be minimal; push complex logic into variables/facts
- Templates are rendered **at runtime** — Ansible renders `.j2` files on the controller before pushing to managed nodes
- All Ansible variable types (vars, facts, hostvars, group_vars) are available inside `{{ }}`

---

## 2. Delimiter Syntax

```jinja2
{# ──────────────────────────────────────────────── #}
{# THREE DELIMITER TYPES                            #}
{# ──────────────────────────────────────────────── #}

{{ expression }}    {# Output — evaluates and prints the value #}
{% statement %}     {# Control — if, for, set, block, macro, etc. #}
{# comment #}       {# Comment — never appears in rendered output #}
```

### Raw blocks — disable Jinja2 processing

```jinja2
{# Use raw/endraw to output literal Jinja2 syntax (e.g., in a Helm chart template) #}
{% raw %}
  {{ .Values.image.tag }}    {# this won't be processed by Jinja2 #}
{% endraw %}
```

### Custom delimiters (Python API — rare in Ansible)

```python
# If your config files already use {{ }}, you can change delimiters
from jinja2 import Environment
env = Environment(
    variable_start_string='<%',
    variable_end_string='%>',
    block_start_string='<%',
    block_end_string='%>',
)
```

---

## 3. Variables & Filters

### Variable access

```jinja2
{{ my_var }}                        {# simple variable #}
{{ my_dict['key'] }}                {# dict by bracket notation #}
{{ my_dict.key }}                   {# dict by dot notation (same result) #}
{{ my_list[0] }}                    {# list index #}
{{ my_dict.nested.deep }}           {# nested dict traversal #}
{{ hostvars['web01']['ansible_host'] }}   {# Ansible host variable #}
```

### Undefined variables

```jinja2
{# Jinja2 raises UndefinedError by default #}
{# Ansible sets undefined_variables=true — access to undefined vars is an error #}

{# Use default filter to provide a fallback #}
{{ my_var | default('fallback_value') }}

{# default(omit) — omit the argument entirely (Ansible-specific) #}
- name: create user
  user:
    name: "{{ username }}"
    shell: "{{ user_shell | default(omit) }}"
```

### Filters — syntax

```jinja2
{# Filters transform a value. Chain them with | pipe operator #}
{{ value | filter_name }}
{{ value | filter_name(arg1, arg2) }}
{{ value | filter1 | filter2 | filter3 }}
```

### String filters

```jinja2
{{ "hello world" | upper }}             {# HELLO WORLD #}
{{ "HELLO" | lower }}                   {# hello #}
{{ "hello world" | title }}             {# Hello World #}
{{ "  hello  " | trim }}               {# hello #}
{{ "hello\n" | trim }}                 {# hello #}
{{ "hello world" | replace("world", "jinja2") }}   {# hello jinja2 #}
{{ my_string | truncate(20) }}          {# truncate to 20 chars with ellipsis #}
{{ my_string | truncate(20, False) }}   {# truncate at word boundary #}
{{ "hello" | center(20) }}             {# "       hello        " #}
{{ "hello" | ljust(10) }}              {# "hello     " #}
{{ "hello" | rjust(10) }}              {# "     hello" #}
{{ my_var | string }}                   {# cast to string #}
{{ my_var | wordcount }}               {# count words in string #}
{{ "2024-01-15" | regex_replace('^(\d{4})-(\d{2})-(\d{2})$', '\3/\2/\1') }}
```

### List/collection filters

```jinja2
{{ [3, 1, 2] | sort }}                 {# [1, 2, 3] #}
{{ [3, 1, 2] | sort(reverse=True) }}   {# [3, 2, 1] #}
{{ ["b","a","c"] | sort }}             {# ["a","b","c"] #}
{{ my_list | reverse | list }}         {# reversed list #}
{{ my_list | unique }}                 {# deduplicate #}
{{ my_list | flatten }}                {# flatten nested lists one level #}
{{ my_list | flatten(levels=2) }}      {# flatten 2 levels deep #}
{{ [1, 2] + [3, 4] }}                  {# concatenate [1,2,3,4] #}
{{ my_list | length }}                 {# count items #}
{{ my_list | count }}                  {# alias for length #}
{{ my_list | first }}                  {# first element #}
{{ my_list | last }}                   {# last element #}
{{ my_list | min }}                    {# minimum value #}
{{ my_list | max }}                    {# maximum value #}
{{ my_list | sum }}                    {# sum of numeric list #}
{{ my_list | join(", ") }}             {# join with separator: "a, b, c" #}
{{ my_list | join("\n") }}             {# join with newline #}
{{ my_list | random }}                 {# random element #}
{{ my_list | shuffle }}                {# random order (Ansible) #}
{{ my_list | zip(other_list) | list }} {# zip two lists #}
{{ my_list | map('upper') | list }}    {# apply filter to each item #}
{{ my_list | select('match', '^web') | list }}   {# filter items matching regex #}
{{ my_list | reject('match', '^db') | list }}    {# reject items matching regex #}
{{ my_list | selectattr('state', 'eq', 'active') | list }}  {# filter by attribute #}
{{ my_list | map(attribute='name') | list }}     {# pluck attribute from objects #}
```

### Dict filters

```jinja2
{{ my_dict | dict2items }}             {# convert dict to list of {key, value} dicts #}
{{ my_list | items2dict }}             {# reverse: list of {key,value} → dict #}
{{ my_dict | combine(other_dict) }}    {# merge dicts (other_dict wins on collision) #}
{{ my_dict | combine(other_dict, recursive=True) }}   {# deep merge #}
{{ my_dict.keys() | list }}            {# list of keys #}
{{ my_dict.values() | list }}          {# list of values #}
{{ my_dict.items() | list }}           {# list of (key, value) tuples #}
```

### Type conversion filters

```jinja2
{{ "42" | int }}                       {# 42 (integer) #}
{{ "3.14" | float }}                   {# 3.14 #}
{{ 42 | string }}                      {# "42" #}
{{ "true" | bool }}                    {# True #}
{{ my_var | list }}                    {# convert to list #}
{{ my_var | set }}                     {# convert to set (unique values) #}
```

### Encoding / hashing filters (Ansible)

```jinja2
{{ "hello" | b64encode }}             {# base64 encode #}
{{ encoded_var | b64decode }}          {# base64 decode #}
{{ "password" | password_hash('sha512') }}    {# hash for /etc/shadow #}
{{ "password" | password_hash('sha512', 'mysalt') }}
{{ my_var | to_json }}                 {# serialize to JSON string #}
{{ my_var | to_nice_json }}            {# pretty-printed JSON #}
{{ my_var | to_nice_json(indent=4) }}
{{ my_json_string | from_json }}       {# parse JSON string → object #}
{{ my_var | to_yaml }}                 {# serialize to YAML string #}
{{ my_yaml_string | from_yaml }}       {# parse YAML string → object #}
{{ my_yaml_string | from_yaml_all | list }}  {# parse multi-doc YAML #}
{{ my_string | hash('sha1') }}         {# SHA1 hash #}
{{ my_string | hash('md5') }}
{{ my_string | checksum }}             {# SHA1 checksum (Ansible) #}
```

### Path / URL filters (Ansible)

```jinja2
{{ "/etc/nginx/nginx.conf" | basename }}    {# nginx.conf #}
{{ "/etc/nginx/nginx.conf" | dirname }}     {# /etc/nginx #}
{{ "/etc/nginx" | dirname }}               {# /etc #}
{{ "nginx.conf" | splitext }}              {# ["nginx", ".conf"] #}
{{ my_path | expanduser }}                 {# expand ~ to home dir #}
{{ my_path | realpath }}                   {# resolve symlinks #}
{{ "/etc" | relpath("/") }}               {# etc #}
{{ my_string | urlencode }}               {# URL-encode a string #}
{{ my_string | urlsplit('hostname') }}     {# extract URL part #}
```

### Networking filters (Ansible)

```jinja2
{{ "192.168.1.5/24" | ipaddr }}           {# validate/normalize IP #}
{{ "192.168.1.5/24" | ipaddr('address') }} {# 192.168.1.5 #}
{{ "192.168.1.5/24" | ipaddr('prefix') }}  {# 24 #}
{{ "192.168.1.5/24" | ipaddr('network') }} {# 192.168.1.0/24 #}
{{ "192.168.1.5/24" | ipaddr('netmask') }} {# 255.255.255.0 #}
{{ "192.168.1.5" | ipaddr('bool') }}       {# True (is valid IP?) #}
{{ my_cidr | ipsubnet(26, 0) }}            {# get first /26 subnet #}
{{ my_ip | ipwrap }}                       {# wrap IPv6 in brackets #}
```

---

## 4. Control Structures

### If / elif / else

```jinja2
{% if env == "production" %}
max_connections = 500
{% elif env == "staging" %}
max_connections = 100
{% else %}
max_connections = 10
{% endif %}

{# Inline conditional (ternary-like) #}
{{ "enabled" if feature_flag else "disabled" }}
{{ value if value is defined else "default" }}

{# Complex conditions #}
{% if env == "production" and replicas >= 3 %}
ha_mode = true
{% endif %}

{% if env not in ["production", "staging"] %}
debug = true
{% endif %}
```

### For loops

```jinja2
{# Iterate over a list #}
{% for server in servers %}
  - {{ server }}
{% endfor %}

{# Iterate over a dict #}
{% for key, value in my_dict.items() %}
{{ key }}: {{ value }}
{% endfor %}

{# Loop with index #}
{% for item in items %}
{{ loop.index }}. {{ item }}       {# 1-based index #}
{{ loop.index0 }}. {{ item }}      {# 0-based index #}
{% endfor %}

{# Loop special variables #}
{% for item in items %}
  loop.index     = {{ loop.index }}      {# 1-based position #}
  loop.index0    = {{ loop.index0 }}     {# 0-based position #}
  loop.revindex  = {{ loop.revindex }}   {# from end, 1-based #}
  loop.revindex0 = {{ loop.revindex0 }}  {# from end, 0-based #}
  loop.first     = {{ loop.first }}      {# True if first iteration #}
  loop.last      = {{ loop.last }}       {# True if last iteration #}
  loop.length    = {{ loop.length }}     {# total count #}
  loop.depth     = {{ loop.depth }}      {# nesting depth, 1-based #}
  loop.depth0    = {{ loop.depth0 }}     {# nesting depth, 0-based #}
{% endfor %}

{# Loop with else (runs if list is empty) #}
{% for server in servers %}
  {{ server }}
{% else %}
  # no servers defined
{% endfor %}

{# Loop with filter #}
{% for user in users | selectattr('active') %}
{{ user.name }}
{% endfor %}

{# Nested loops #}
{% for host in groups['webservers'] %}
  {% for port in [80, 443] %}
  {{ host }}:{{ port }}
  {% endfor %}
{% endfor %}
```

### Set — assign variables

```jinja2
{# Assign within template #}
{% set my_var = "value" %}
{% set count = 0 %}
{% set servers = ["web1", "web2", "web3"] %}

{# Build a list incrementally (requires namespace trick) #}
{% set ns = namespace(items=[]) %}
{% for item in raw_list %}
  {% if item.active %}
    {% set ns.items = ns.items + [item.name] %}
  {% endif %}
{% endfor %}
{{ ns.items | join(", ") }}

{# Dict building #}
{% set config = {
  'host': db_host,
  'port': db_port,
  'name': db_name
} %}
```

### Block assignments (multi-line capture)

```jinja2
{# Capture a block of rendered text into a variable #}
{% set my_block %}
  server {
    listen 80;
    server_name {{ domain }};
  }
{% endset %}
{{ my_block }}
```

---

## 5. Tests

Tests check properties of values. Used with `is` and `is not`.

```jinja2
{# Defined / undefined #}
{% if my_var is defined %}
{% if my_var is not defined %}
{% if my_var is undefined %}

{# None / null check #}
{% if my_var is none %}
{% if my_var is not none %}

{# Type checks #}
{% if my_var is string %}
{% if my_var is number %}
{% if my_var is integer %}
{% if my_var is float %}
{% if my_var is boolean %}
{% if my_var is iterable %}
{% if my_var is mapping %}           {# is a dict #}
{% if my_var is sequence %}          {# is a list or string #}

{# Truth checks #}
{% if my_var is truthy %}
{% if my_var is falsy %}

{# Numeric tests #}
{% if count is even %}
{% if count is odd %}
{% if count is divisibleby(3) %}

{# String tests #}
{% if my_var is match("^web") %}     {# regex match from start (Ansible) #}
{% if my_var is search("nginx") %}   {# regex search anywhere (Ansible) #}

{# Collection tests #}
{% if my_list is subset(other_list) %}    {# Ansible: all items in other_list #}
{% if my_list is superset(other_list) %}  {# Ansible: contains all of other_list #}
{% if item is in my_list %}               {# membership test #}

{# Ansible-specific #}
{% if my_var is vault_encrypted %}   {# check if value is Ansible Vault encrypted #}
{% if my_var is abs %}               {# absolute value check #}
```

---

## 6. Macros & Includes

### Macros — reusable template functions

```jinja2
{# Define a macro #}
{% macro render_server(name, port=80, ssl=False) %}
server {
  listen {{ port }}{% if ssl %}s{% endif %};
  server_name {{ name }};
}
{% endmacro %}

{# Call the macro #}
{{ render_server("api.example.com") }}
{{ render_server("app.example.com", port=443, ssl=True) }}

{# Macro with caller (block content) #}
{% macro render_block(title) %}
## {{ title }}
{{ caller() }}
{% endmacro %}

{% call render_block("My Section") %}
  This content is passed as caller().
{% endcall %}

{# Access macro metadata #}
{{ render_server.name }}         {# "render_server" #}
{{ render_server.arguments }}    {# list of argument names #}
{{ render_server.defaults }}     {# dict of defaults #}
```

### Import — use macros from another file

```jinja2
{# macros/network.j2 contains macro definitions #}

{# Import all macros from a file (with namespace) #}
{% import "macros/network.j2" as net %}
{{ net.render_server("web1") }}

{# Import specific macros into current namespace #}
{% from "macros/network.j2" import render_server, render_upstream %}
{{ render_server("web1") }}

{# Import without caching (re-renders every time) #}
{% import "macros/network.j2" as net without context %}
```

### Include — insert another template

```jinja2
{# Include a partial template — shares current context #}
{% include "partials/header.j2" %}

{# Include with ignore missing (no error if file not found) #}
{% include "partials/optional.j2" ignore missing %}

{# Include from a list — first one found wins #}
{% include ["custom_header.j2", "partials/header.j2"] ignore missing %}
```

---

## 7. Inheritance & Blocks

### Base template (`base.j2`)

```jinja2
{# base.j2 — skeleton with named blocks to override #}
# Managed by Ansible — DO NOT EDIT MANUALLY
# Generated: {{ ansible_date_time.date }}
# Host: {{ inventory_hostname }}

{% block global_config %}
# Global defaults
worker_processes auto;
{% endblock %}

{% block main_config %}
{# subclasses override this #}
{% endblock %}

{% block includes %}
include /etc/nginx/conf.d/*.conf;
{% endblock %}
```

### Child template

```jinja2
{# nginx.conf.j2 — extends base.j2 #}
{% extends "base.j2" %}

{% block global_config %}
{# Call parent block content first, then add #}
{{ super() }}
worker_rlimit_nofile 65535;
{% endblock %}

{% block main_config %}
http {
  sendfile on;
  server_tokens off;
  {% for vhost in virtual_hosts %}
  include /etc/nginx/sites-enabled/{{ vhost }}.conf;
  {% endfor %}
}
{% endblock %}

{# Block not overridden → inherits base.j2 block content #}
```

---

## 8. Whitespace Control

Jinja2 preserves all whitespace by default. Control it with `-` modifiers.

```jinja2
{# ─────────────────────────────────────── #}
{# WHITESPACE TRIM MODIFIERS              #}
{#  {%- strips whitespace BEFORE the tag #}
{#  -%} strips whitespace AFTER the tag  #}
{# ─────────────────────────────────────── #}

{# Without control — produces blank lines around if blocks #}
{% if condition %}
value = true
{% endif %}

{# With trim — no extra blank lines #}
{%- if condition %}
value = true
{%- endif %}

{# Trim on both sides #}
{%- for item in items -%}
{{ item }}
{%- endfor -%}

{# Trim only end of tag (most common in config files) #}
{% for item in items -%}
  - {{ item }}
{% endfor %}
```

### `trim_blocks` and `lstrip_blocks` (environment settings)

```python
# Python API — most useful for non-Ansible usage
from jinja2 import Environment
env = Environment(
    trim_blocks=True,    # Remove first newline after block tags
    lstrip_blocks=True,  # Strip leading whitespace from block tags
)
```

```ini
# Ansible sets these via ansible.cfg
[defaults]
jinja2_trim_blocks   = True   # equivalent to trim_blocks=True
jinja2_lstrip_blocks = True   # equivalent to lstrip_blocks=True
```

---

## 9. Ansible — Jinja2 in Playbooks

### Variable interpolation in tasks

```yaml
# All {{ }} expressions are Jinja2 — evaluated at task runtime
- name: start nginx
  service:
    name: "{{ service_name }}"          # variable
    state: "{{ 'started' if start_service else 'stopped' }}"  # conditional

# YAML gotcha: quote strings starting with {{ }}
- name: set fact
  set_fact:
    url: "{{ base_url }}/api/v1"       # ✅ quoted
    # url: {{ base_url }}/api/v1       # ❌ YAML parse error — must quote
```

### `when` — conditional task execution

```yaml
# when uses Jinja2 expressions WITHOUT {{ }}
- name: install on RedHat
  yum:
    name: nginx
    state: present
  when: ansible_os_family == "RedHat"

- name: install on Debian
  apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"

# Multiple conditions — AND (list syntax)
- name: restart in production
  service:
    name: myapp
    state: restarted
  when:
    - env == "production"
    - service_running is defined
    - service_running | bool

# OR condition
- name: on web or app servers
  debug:
    msg: "web or app"
  when: "'web' in group_names or 'app' in group_names"

# Check if variable is defined and not empty
- name: only if var set
  debug:
    msg: "{{ my_var }}"
  when: my_var is defined and my_var | length > 0

# Negate
- name: skip on containers
  service:
    name: cron
    state: started
  when: not ansible_virtualization_type == "docker"
```

### `register` and result inspection

```yaml
- name: check if config exists
  stat:
    path: /etc/myapp/config.yaml
  register: config_stat

- name: create config if missing
  template:
    src: config.yaml.j2
    dest: /etc/myapp/config.yaml
  when: not config_stat.stat.exists

- name: run command
  command: /usr/bin/myapp --check
  register: check_result
  ignore_errors: true

- name: show output
  debug:
    msg: "Exit code: {{ check_result.rc }}, stdout: {{ check_result.stdout }}"

- name: fail if unexpected output
  fail:
    msg: "Unexpected output: {{ check_result.stdout }}"
  when: "'ERROR' in check_result.stdout"
```

### `set_fact` — compute and store variables

```yaml
- name: compute derived facts
  set_fact:
    app_url:       "https://{{ ansible_fqdn }}:{{ app_port }}"
    is_primary:    "{{ inventory_hostname == groups['db'][0] }}"
    worker_count:  "{{ ansible_processor_vcpus * 2 }}"
    db_hosts:      "{{ groups['db'] | map('extract', hostvars, 'ansible_host') | list }}"
    cacheable: true    # persist across plays in same playbook run
```

### `vars` inline and `vars_files`

```yaml
# Inline vars (highest precedence in a play)
- hosts: webservers
  vars:
    nginx_worker_processes: 4
    nginx_worker_connections: 1024
    ssl_protocols: ["TLSv1.2", "TLSv1.3"]

  vars_files:
    - vars/common.yaml
    - "vars/{{ env }}.yaml"    # dynamic file selection per environment
```

---

## 10. Ansible — template Module & Files

### `template` module

```yaml
- name: render nginx config
  template:
    src: nginx.conf.j2       # relative to role/files/ or playbook/templates/
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
    backup: true             # keep backup of previous file
    validate: "/usr/sbin/nginx -t -c %s"   # validate before replacing
  notify: reload nginx
```

### Template file (`templates/nginx.conf.j2`)

```jinja2
{# templates/nginx.conf.j2 #}
# Managed by Ansible — DO NOT EDIT MANUALLY
# Template: {{ template_path | basename }}
# Generated: {{ ansible_date_time.iso8601 }}

user {{ nginx_user | default('www-data') }};
worker_processes {{ nginx_worker_processes | default(ansible_processor_vcpus) }};
worker_rlimit_nofile {{ nginx_worker_rlimit | default(65535) }};

error_log /var/log/nginx/error.log {{ nginx_error_log_level | default('warn') }};
pid /run/nginx.pid;

events {
  worker_connections {{ nginx_worker_connections | default(1024) }};
  use epoll;
  multi_accept on;
}

http {
  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;
  keepalive_timeout {{ nginx_keepalive_timeout | default(65) }};
  types_hash_max_size 2048;
  server_tokens off;

  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  {% if nginx_log_format is defined %}
  log_format main '{{ nginx_log_format }}';
  {% endif %}
  access_log /var/log/nginx/access.log{% if nginx_log_format is defined %} main{% endif %};

  {% if ssl_certificate is defined %}
  ssl_certificate     {{ ssl_certificate }};
  ssl_certificate_key {{ ssl_certificate_key }};
  ssl_protocols       {{ ssl_protocols | join(' ') }};
  ssl_ciphers         {{ ssl_ciphers | default('HIGH:!aNULL:!MD5') }};
  ssl_prefer_server_ciphers on;
  {% endif %}

  {% for upstream in nginx_upstreams | default([]) %}
  upstream {{ upstream.name }} {
    {% if upstream.method is defined %}
    {{ upstream.method }};
    {% endif %}
    {% for server in upstream.servers %}
    server {{ server }}{% if upstream.weight is defined %} weight={{ upstream.weight }}{% endif %};
    {% endfor %}
    keepalive {{ upstream.keepalive | default(32) }};
  }

  {% endfor %}
  {% for vhost in nginx_vhosts | default([]) %}
  server {
    listen {{ vhost.port | default(80) }}{% if vhost.default_server | default(False) %} default_server{% endif %};
    {% if vhost.ssl | default(False) %}
    listen {{ vhost.ssl_port | default(443) }} ssl http2;
    {% endif %}
    server_name {{ vhost.server_name | join(' ') }};
    root {{ vhost.root | default('/var/www/html') }};

    {% if vhost.return_redirect is defined %}
    return {{ vhost.return_redirect.code }} {{ vhost.return_redirect.url }};
    {% else %}
    location / {
      proxy_pass http://{{ vhost.upstream }};
      proxy_set_header Host              $host;
      proxy_set_header X-Real-IP         $remote_addr;
      proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
    {% endif %}
  }
  {% endfor %}
}
```

### Rendering `lookup` inline (alternative to template file)

```yaml
# Render a Jinja2 string inline using lookup
- name: render inline
  debug:
    msg: "{{ lookup('template', 'templates/message.j2') }}"

# Use vars from a YAML/JSON file
- name: read external vars
  set_fact:
    db_config: "{{ lookup('file', 'configs/db.json') | from_json }}"

# Render with pipe template (Ansible 2.9+)
- name: render config string
  copy:
    content: "{{ lookup('template', 'templates/config.j2') }}"
    dest: /etc/myapp/config.conf
```

---

## 11. Ansible — Filters Deep Dive

### Combining and manipulating data

```yaml
vars:
  base_packages:
    - curl
    - git
    - vim

  extra_packages:
    - htop
    - jq

  # Combine lists
  all_packages: "{{ base_packages + extra_packages }}"

  # Unique items only
  unique_packages: "{{ (base_packages + extra_packages) | unique }}"

  # Difference (items in A not in B)
  to_remove: "{{ installed | difference(desired) }}"

  # Intersection (items in both)
  common: "{{ list_a | intersect(list_b) }}"

  # Union (all items from both, deduplicated)
  all_items: "{{ list_a | union(list_b) }}"

  # Symmetric difference (in one but not both)
  only_in_one: "{{ list_a | symmetric_difference(list_b) }}"
```

### Working with dicts

```yaml
vars:
  defaults:
    timeout: 30
    retries: 3
    log_level: info

  overrides:
    timeout: 60
    log_level: debug

  # Merge: overrides win
  config: "{{ defaults | combine(overrides) }}"
  # Result: {timeout: 60, retries: 3, log_level: debug}

  # Deep merge (nested dicts)
  merged: "{{ base_config | combine(env_config, recursive=True) }}"

  # Convert dict to list of {key, value}
  config_items: "{{ config | dict2items }}"
  # [{key: timeout, value: 60}, {key: retries, value: 3}, ...]

  # Filter dict items
  only_long: "{{ config | dict2items | selectattr('value', 'gt', 10) | items2dict }}"
```

### Extracting data from complex structures

```yaml
vars:
  servers:
    - name: web01
      ip: 10.0.0.1
      tags: [web, nginx]
    - name: db01
      ip: 10.0.0.10
      tags: [db, postgres]
    - name: web02
      ip: 10.0.0.2
      tags: [web, nginx]

  # Extract a single attribute
  all_names: "{{ servers | map(attribute='name') | list }}"
  # [web01, db01, web02]

  # Filter by attribute value
  web_servers: "{{ servers | selectattr('name', 'match', '^web') | list }}"

  # Chain: filter then extract
  web_ips: "{{ servers | selectattr('name', 'match', '^web') | map(attribute='ip') | list }}"
  # [10.0.0.1, 10.0.0.2]

  # Group by attribute
  by_role: "{{ servers | groupby('role') }}"

  # Use extract to look up hostvars
  web_ansible_hosts: >-
    {{ groups['web'] | map('extract', hostvars, 'ansible_host') | list }}
```

### JSON query filter (`json_query`)

```yaml
# json_query uses JMESPath syntax
vars:
  services:
    - name: nginx
      port: 80
      state: running
    - name: postgres
      port: 5432
      state: stopped
    - name: redis
      port: 6379
      state: running

  # Select names of running services
  running_names: "{{ services | json_query('[?state==`running`].name') }}"
  # ["nginx", "redis"]

  # Get all ports
  all_ports: "{{ services | json_query('[*].port') }}"
  # [80, 5432, 6379]

  # Filter and project
  running_services: "{{ services | json_query('[?state==`running`].{name: name, port: port}') }}"
```

### String manipulation for config generation

```yaml
vars:
  # Indent a multiline string (useful for YAML embedding)
  indented_config: "{{ raw_config | indent(4) }}"
  indented_first:  "{{ raw_config | indent(4, first=True) }}"

  # Regex operations
  version_string: "v1.23.4"
  version_only: "{{ version_string | regex_replace('^v', '') }}"
  # 1.23.4

  major_version: "{{ version_string | regex_search('(\\d+)\\.', '\\1') | first }}"
  # "1"

  all_versions: "{{ text | regex_findall('v\\d+\\.\\d+\\.\\d+') }}"

  # Format strings
  padded: "{{ 42 | string | zfill(5) }}"   # "00042"
```

---

## 12. Ansible — Conditionals & Loops

### `loop` (modern, replaces `with_*`)

```yaml
# Loop over a list
- name: install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - curl
    - git

# Loop over a list variable
- name: create directories
  file:
    path: "{{ item }}"
    state: directory
    mode: "0755"
  loop: "{{ app_directories }}"

# Loop over dict items
- name: set sysctl params
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    state: present
  loop: "{{ sysctl_params | dict2items }}"

# Loop with index
- name: create numbered files
  copy:
    content: "server {{ idx + 1 }}"
    dest: "/etc/servers/server{{ idx + 1 }}.conf"
  loop: "{{ server_list }}"
  loop_control:
    index_var: idx
    loop_var: server     # rename 'item' to avoid conflicts in nested loops
    label: "{{ server }}"  # cleaner output in --verbose mode

# Loop with subelements (nested)
- name: add authorized keys per user
  authorized_key:
    user: "{{ item.0.name }}"
    key:  "{{ item.1 }}"
  loop: "{{ users | subelements('ssh_keys') }}"

# Loop with product (cartesian)
- name: configure listen ports
  template:
    src: listener.conf.j2
    dest: "/etc/app/listener-{{ item.0 }}-{{ item.1 }}.conf"
  loop: "{{ ['http', 'https'] | product(['8080', '8443']) | list }}"

# Retry loop
- name: wait for service
  uri:
    url: "http://{{ ansible_host }}:{{ app_port }}/health"
    status_code: 200
  register: health_check
  until: health_check.status == 200
  retries: 10
  delay: 5
```

### `block` — group tasks with shared attributes

```yaml
- name: install and configure on production
  block:
    - name: install app
      apt:
        name: myapp
        state: present

    - name: deploy config
      template:
        src: myapp.conf.j2
        dest: /etc/myapp/myapp.conf

    - name: start service
      service:
        name: myapp
        state: started
        enabled: true

  when: env == "production"
  become: true
  tags: [myapp, install]
  rescue:
    - name: rollback on failure
      command: myapp-rollback
  always:
    - name: send notification
      uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: "Deployment {{ 'succeeded' if not ansible_failed_task else 'FAILED' }}"
```

---

## 13. Ansible — Roles & Variable Precedence

### Role structure

```
roles/
└── nginx/
    ├── defaults/
    │   └── main.yaml       # lowest precedence — default values
    ├── vars/
    │   └── main.yaml       # higher precedence — role-internal constants
    ├── tasks/
    │   ├── main.yaml       # entry point
    │   ├── install.yaml
    │   └── configure.yaml
    ├── handlers/
    │   └── main.yaml       # triggered by notify:
    ├── templates/
    │   └── nginx.conf.j2   # Jinja2 templates
    ├── files/
    │   └── static.conf     # static files (no templating)
    ├── meta/
    │   └── main.yaml       # role dependencies
    └── README.md
```

### Variable precedence (lowest → highest)

```
1.  role defaults          (roles/myrole/defaults/main.yaml)
2.  inventory file vars    (host_vars, group_vars in inventory)
3.  inventory group_vars   (group_vars/all.yaml, group_vars/webservers.yaml)
4.  inventory host_vars    (host_vars/web01.yaml)
5.  playbook group_vars    (playbook-level group_vars/)
6.  playbook host_vars     (playbook-level host_vars/)
7.  host facts             (gathered by setup module)
8.  play vars              (vars: in a play)
9.  play vars_prompt
10. play vars_files
11. role vars              (roles/myrole/vars/main.yaml)
12. block vars
13. task vars              (vars: on a task)
14. include_vars
15. set_facts / registered vars
16. role/include params
17. extra vars             (-e on command line)   ← HIGHEST
```

### `group_vars` / `host_vars`

```
inventory/
├── hosts.ini
├── group_vars/
│   ├── all.yaml              # applies to all hosts
│   ├── all/
│   │   ├── common.yaml       # split across multiple files
│   │   └── vault.yaml        # encrypted secrets
│   ├── webservers.yaml       # applies to [webservers] group
│   └── production/
│       ├── vars.yaml
│       └── vault.yaml
└── host_vars/
    ├── web01.yaml
    └── db01.yaml
```

```yaml
# group_vars/webservers.yaml
nginx_worker_processes: 4
nginx_worker_connections: 2048
app_port: 8080
ssl_enabled: true

# host_vars/web01.yaml
nginx_worker_processes: 8    # overrides group var for this host only
ansible_host: 10.0.0.1
ansible_user: ubuntu
```

### Handlers

```yaml
# handlers/main.yaml
- name: reload nginx
  service:
    name: nginx
    state: reloaded

- name: restart nginx
  service:
    name: nginx
    state: restarted

- name: restart postgres
  service:
    name: postgresql
    state: restarted

# tasks/configure.yaml — notify handlers
- name: deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: reload nginx           # triggers handler at end of play

- name: deploy ssl cert
  copy:
    src: cert.pem
    dest: /etc/ssl/certs/app.pem
  notify:
    - reload nginx
    - restart nginx              # multiple handlers
```

---

## 14. SaltStack — Jinja2 in States

### State file with Jinja2 (`nginx/init.sls`)

```jinja2
{# Salt makes grains, pillar, and salt functions available #}
{%- set nginx_user = salt['pillar.get']('nginx:user', 'www-data') %}
{%- set is_rhel = grains['os_family'] == 'RedHat' %}

nginx_package:
  pkg.installed:
    - name: {{ salt['pillar.get']('nginx:package', 'nginx') }}

nginx_config:
  file.managed:
    - name: /etc/nginx/nginx.conf
    - source: salt://nginx/files/nginx.conf.j2
    - template: jinja
    - user: root
    - group: root
    - mode: 644
    - context:
        nginx_user: {{ nginx_user }}
        worker_processes: {{ grains['num_cpus'] }}

nginx_service:
  service.running:
    - name: nginx
    - enable: true
    - watch:
      - file: nginx_config
```

### Pillar data access

```jinja2
{# Salt Jinja2 specific functions #}
{{ pillar['myapp']['db_host'] }}                    {# direct access (KeyError if missing) #}
{{ salt['pillar.get']('myapp:db_host', 'localhost') }}   {# safe with default #}
{{ salt['grains.get']('os_family', 'Debian') }}          {# grains with default #}
{{ salt['config.get']('myapp:workers', 4) }}             {# config with default #}

{# Grain-based conditionals #}
{%- if grains['os'] == 'Ubuntu' %}
service_name: nginx
{%- elif grains['os'] == 'CentOS' %}
service_name: nginx
{%- endif %}

{# Loop over pillar data #}
{%- for user, data in pillar.get('users', {}).items() %}
user_{{ user }}:
  user.present:
    - name: {{ user }}
    - shell: {{ data.get('shell', '/bin/bash') }}
    - groups: {{ data.get('groups', []) | tojson }}
{%- endfor %}
```

---

## 15. Common Gotchas & Best Practices

### YAML + Jinja2 quoting rules

```yaml
# ALWAYS quote values starting with {{ }}
# YAML will try to parse bare {{ as a flow mapping

# ❌ WRONG — YAML parse error
- name: bad
  debug:
    msg: {{ my_var }}

# ✅ CORRECT
- name: good
  debug:
    msg: "{{ my_var }}"

# ❌ WRONG — multiline value needs quoting too
- name: bad
  set_fact:
    url: {{ base_url }}/path

# ✅ CORRECT
- name: good
  set_fact:
    url: "{{ base_url }}/path"
```

### Boolean coercion pitfalls

```yaml
# Ansible YAML booleans: yes/no/true/false/on/off (case insensitive)
# Jinja2 | bool filter understands "yes", "true", "1", "on" → True

vars:
  # These are all YAML booleans (not strings):
  a: true
  b: yes
  c: on

  # These become string "True" in Jinja2 without | bool
  from_env: "{{ lookup('env', 'MY_FLAG') | bool }}"

# Always use | bool when the source might be a string
when: my_flag | bool
```

### Undefined variable handling

```yaml
# Ansible default: error on undefined (unlike raw Jinja2 which silently outputs "")
# Use | default() to make variables optional

vars:
  # Safe access with default
  log_level: "{{ app_log_level | default('info') }}"

  # Default to omit (skip the argument entirely in a module)
  optional_timeout: "{{ custom_timeout | default(omit) }}"

  # Nested safe access
  db_port: "{{ db_config.port | default(5432) }}"

  # Check before using
  ssl_cert: "{{ ssl_certificate if ssl_certificate is defined else '' }}"
```

### `loop` vs `with_*` — use `loop`

```yaml
# DEPRECATED — avoid with_* directives in new code
- name: old style
  apt:
    name: "{{ item }}"
  with_items:
    - nginx
    - curl

# CORRECT — modern loop directive
- name: new style
  apt:
    name: "{{ item }}"
  loop:
    - nginx
    - curl

# with_dict → loop + dict2items
# OLD:
- debug:
    msg: "{{ item.key }}: {{ item.value }}"
  with_dict: "{{ my_dict }}"

# NEW:
- debug:
    msg: "{{ item.key }}: {{ item.value }}"
  loop: "{{ my_dict | dict2items }}"
```

### Idempotency and template comments

```jinja2
{# Include a generated-by header in ALL managed files #}
# This file is managed by Ansible.
# Manual changes will be overwritten on next playbook run.
# Source: {{ role_path | basename }}/templates/{{ template_path | basename }}
# Last rendered: {{ ansible_date_time.iso8601 }}
```

### `no_log` for sensitive values

```yaml
# Prevent secrets from appearing in output or logs
- name: set db password
  lineinfile:
    path: /etc/myapp/db.conf
    line: "password={{ db_password }}"
  no_log: true

# Also suppress loop output for sensitive loops
- name: create user passwords
  user:
    name: "{{ item.name }}"
    password: "{{ item.password | password_hash('sha512') }}"
  loop: "{{ users }}"
  no_log: true
```

### Jinja2 in `ansible.cfg`

```ini
[defaults]
jinja2_trim_blocks   = True   # Remove newline after block tags
jinja2_lstrip_blocks = True   # Strip leading spaces from block tags
# Prevents extra blank lines in rendered templates

# Enable native types (return actual Python types, not strings)
jinja2_native = False   # Default. Set True only if you need non-string return types
```

### Template testing strategy

```bash
# Dry run — see what would change without applying
ansible-playbook site.yaml --check --diff

# Show rendered template diff
ansible-playbook site.yaml --diff

# Template-only run (tags)
ansible-playbook site.yaml --tags templates

# Test a single template render (quick debug)
ansible -i inventory web01 -m template \
  -a "src=templates/nginx.conf.j2 dest=/tmp/nginx.conf.test" --check --diff

# Debug a variable value
ansible -i inventory web01 -m debug -a "var=nginx_vhosts"
```

---

## 16. Quick Reference Cheat Sheet

```jinja2
{# ─── DELIMITERS ─────────────────────────────────────────────────────────── #}
{{ expression }}        {# output — print value #}
{% statement %}         {# control — if, for, set, macro, block, include #}
{# comment #}           {# comment — stripped from output #}
{% raw %} ... {% endraw %}   {# disable Jinja2 processing inside #}

{# ─── VARIABLE ACCESS ────────────────────────────────────────────────────── #}
{{ var }}                             {# simple variable #}
{{ dict.key }} / {{ dict['key'] }}    {# dict access (equivalent) #}
{{ list[0] }}                         {# list index #}
{{ nested.a.b.c }}                    {# deep traversal #}
{{ hostvars['host']['var'] }}         {# Ansible cross-host var #}

{# ─── COMMON FILTERS ─────────────────────────────────────────────────────── #}
| default('val')       | default(omit)       | bool
| int                  | float               | string
| upper  | lower  | title  | trim  | replace(a,b)
| length | count  | first  | last  | min | max | sum
| sort   | unique | reverse | list  | join(', ')
| flatten          | flatten(levels=N)
| map(attribute='name')    | map('filter')
| select('test')           | reject('test')
| selectattr('k','eq','v') | rejectattr('k','eq','v')
| groupby('attr')
| combine(other)           | combine(other, recursive=True)
| dict2items               | items2dict
| difference(b)    | intersect(b)   | union(b)
| to_json | to_nice_json | from_json | to_yaml | from_yaml
| b64encode | b64decode
| password_hash('sha512')
| regex_replace(pat, repl)   | regex_search(pat)   | regex_findall(pat)
| basename | dirname | expanduser
| ipaddr | ipaddr('address') | ipaddr('network')
| json_query('jmespath_expr')
| indent(4) | indent(4, first=True)

{# ─── TESTS ───────────────────────────────────────────────────────────────── #}
is defined / is not defined / is undefined
is none / is not none
is string / is number / is integer / is float / is boolean
is iterable / is mapping / is sequence
is truthy / is falsy
is even / is odd / is divisibleby(N)
is match('regex') / is search('regex')        {# Ansible #}
is subset(list) / is superset(list)           {# Ansible #}

{# ─── CONTROL FLOW ────────────────────────────────────────────────────────── #}
{% if condition %} ... {% elif cond %} ... {% else %} ... {% endif %}
{% for item in list %} ... {% else %} ... {% endfor %}
{% for k, v in dict.items() %} ... {% endfor %}
{% set var = value %}
{% set ns = namespace(items=[]) %}   {# mutable namespace in loops #}

{# ─── LOOP VARIABLES ─────────────────────────────────────────────────────── #}
loop.index    loop.index0    loop.revindex    loop.revindex0
loop.first    loop.last      loop.length      loop.depth

{# ─── WHITESPACE CONTROL ─────────────────────────────────────────────────── #}
{%- tag %}   {# strip whitespace BEFORE this tag #}
{% tag -%}   {# strip whitespace AFTER this tag #}

{# ─── MACROS & INCLUDES ─────────────────────────────────────────────────── #}
{% macro name(arg, kwarg=default) %} ... {% endmacro %}
{{ name(value) }}
{% import "file.j2" as ns %}     {{ ns.macro() }}
{% from "file.j2" import macro %}
{% include "partial.j2" %}
{% include "partial.j2" ignore missing %}

{# ─── INHERITANCE ─────────────────────────────────────────────────────────── #}
{% extends "base.j2" %}
{% block name %} ... {% endblock %}
{{ super() }}    {# include parent block content #}
```

### Ansible CLI workflow

```bash
# Check syntax only
ansible-playbook site.yaml --syntax-check

# Dry run with diff (see template changes)
ansible-playbook -i inventory site.yaml --check --diff

# Limit to specific hosts or groups
ansible-playbook -i inventory site.yaml --limit webservers
ansible-playbook -i inventory site.yaml --limit web01,web02

# Run only tagged tasks
ansible-playbook -i inventory site.yaml --tags "nginx,config"
ansible-playbook -i inventory site.yaml --skip-tags deploy

# Pass extra vars (highest precedence)
ansible-playbook -i inventory site.yaml -e "env=production app_version=v1.2.3"
ansible-playbook -i inventory site.yaml -e @extra_vars.yaml

# Debug a variable on a host
ansible -i inventory web01 -m debug -a "var=ansible_all_ipv4_addresses"

# Gather facts only
ansible -i inventory all -m setup

# Gather filtered facts
ansible -i inventory all -m setup -a "filter=ansible_memory_mb"

# Run ad-hoc command
ansible -i inventory webservers -m command -a "nginx -v" --become

# List hosts that would be targeted
ansible-playbook -i inventory site.yaml --list-hosts

# Step through tasks one by one
ansible-playbook -i inventory site.yaml --step
```

### Key rules at a glance

| Rule | Detail |
|------|--------|
| Quote `{{ }}` in YAML | Bare `{{ }}` is a YAML flow mapping — always wrap in `"..."` |
| `loop` over `with_*` | `with_items`, `with_dict` etc. are deprecated; use `loop` |
| `\| default(omit)` | Skip optional module arguments cleanly instead of conditional tasks |
| `\| bool` on string booleans | `"true"` ≠ `True` — always coerce from env vars or string sources |
| `\| combine(recursive=True)` | Deep merge dicts; plain `combine` only merges top-level keys |
| `no_log: true` for secrets | Suppress task output when handling passwords, tokens, keys |
| `--check --diff` always | Preview template changes before applying to production |
| `defaults/` vs `vars/` | `defaults/` = overridable by callers; `vars/` = internal constants |
| Commit `requirements.yaml` | Pin collection and role versions for reproducible runs |
| `namespace()` in loops | Use `ns = namespace(var=value)` to mutate variables inside `for` loops |
| Avoid logic in templates | Move complex Jinja2 into `set_fact` tasks; keep templates readable |

---

*Part of DevOpsNotes / LANGUAGES — see also `02_Groovy_Jenkins.md`, `03_HCL.md`, `05_GoTemplates.md`*