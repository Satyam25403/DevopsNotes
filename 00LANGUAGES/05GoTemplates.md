# Go Templates — DevOps Reference Notes

> **Go Templates** — the templating engine built into the Go standard library (`text/template` and `html/template`). In DevOps, Go templates are the backbone of **Helm** (Kubernetes package manager), **kubectl** output formatting, **Kustomize** (limited), **Consul Template**, **Terraform `templatefile`** (uses HCL expressions, not Go templates), and **Hugo** (static sites). The syntax is compact and explicit — no implicit type coercion, errors on undefined keys by default, and composable pipelines.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Delimiter Syntax & Whitespace](#2-delimiter-syntax--whitespace)
3. [Variables, Types & Pipelines](#3-variables-types--pipelines)
4. [Control Structures](#4-control-structures)
5. [Functions & Sprig](#5-functions--sprig)
6. [Template Definition & Reuse](#6-template-definition--reuse)
7. [Helm — Chart Structure](#7-helm--chart-structure)
8. [Helm — Values & Overrides](#8-helm--values--overrides)
9. [Helm — Template Patterns](#9-helm--template-patterns)
10. [Helm — Named Templates & Helpers](#10-helm--named-templates--helpers)
11. [Helm — Flow Control & Conditionals](#11-helm--flow-control--conditionals)
12. [Helm — Hooks & Tests](#12-helm--hooks--tests)
13. [kubectl — Output Formatting](#13-kubectl--output-formatting)
14. [Consul Template](#14-consul-template)
15. [Common Gotchas & Best Practices](#15-common-gotchas--best-practices)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Core Concepts

```
Go Templates are used by:
  Helm            → Kubernetes manifests (.yaml in templates/)
  kubectl         → -o go-template / --output=jsonpath
  Consul Template → dynamic config from Consul/Vault (.ctmpl files)
  Hugo            → static site generation
  go generate     → code generation in Go projects
```

Go template files contain plain text with embedded **actions** delimited by `{{ }}`:

```gotemplate
{{/* comment */}}
{{ .Value }}                  {{/* output a field */}}
{{ if .Condition }} ... {{ end }}
{{ range .Items }} ... {{ end }}
```

Key differences from Jinja2:

```
Go Templates              Jinja2
──────────────────────    ──────────────────────
{{ .Field }}              {{ variable }}
{{ funcName arg }}        {{ variable | filter }}
{{ arg | funcName }}      {{ variable | filter }}   ← same pipe syntax
{{- trim left            {%- trim left
-}}  trim right -%}       trim right
{{/* comment */}}         {# comment #}
{{ define "name" }}       {% macro name() %}
{{ template "name" . }}   {{ name() }}
No undefined — panics     Silent empty string
```

- Actions are evaluated left to right in a **pipeline** — the output of each stage feeds into the next
- The **dot** (`.`) is the current context — it changes inside `range`, `with`, and `template` calls
- Functions are called with **space-separated args**: `{{ funcName arg1 arg2 }}`, not `funcName(arg1, arg2)`
- Helm adds the **Sprig** library (~70 extra functions) on top of the Go standard template functions

---

## 2. Delimiter Syntax & Whitespace

```gotemplate
{{/* This is a comment — stripped from output */}}

{{ .Value }}          {{/* output expression */}}
{{ funcName .Arg }}   {{/* function call */}}
{{ $var := .Field }}  {{/* variable assignment */}}
{{ if .Cond }} ... {{ else }} ... {{ end }}
{{ range .List }} ... {{ end }}
{{ define "name" }} ... {{ end }}
{{ template "name" . }}
{{ with .Value }} ... {{ end }}
```

### Whitespace trimming

By default Go templates preserve all surrounding whitespace. Use `-` to trim:

```gotemplate
{{- /* trim whitespace/newline BEFORE this action */ -}}
{{- .Value }}       {{/* trim newline before */}}
{{ .Value -}}       {{/* trim newline after */}}
{{- .Value -}}      {{/* trim both sides */}}

{{- if .Enabled }}
enabled: true
{{- end }}

{{- range .Items }}
- {{ . }}
{{- end }}
```

### Practical whitespace in Helm YAML

```gotemplate
{{/* Without trim — extra blank lines break YAML readability */}}
spec:
  {{ if .Values.resources }}
  resources:
    {{ toYaml .Values.resources | indent 4 }}
  {{ end }}

{{/* With trim — clean output */}}
spec:
  {{- if .Values.resources }}
  resources:
    {{- toYaml .Values.resources | nindent 4 }}
  {{- end }}
```

---

## 3. Variables, Types & Pipelines

### The dot — current context

```gotemplate
{{/* Top-level: . is the data passed to the template */}}
{{ .Release.Name }}
{{ .Values.image.tag }}

{{/* Inside range: . becomes each element */}}
{{ range .Values.servers }}
  - {{ .host }}:{{ .port }}    {{/* . is now a server object */}}
{{ end }}

{{/* Access outer scope with $. */}}
{{ range .Values.servers }}
  {{ $.Release.Name }}-{{ .host }}    {{/* $. always refers to top-level */}}
{{ end }}
```

### Variable assignment

```gotemplate
{{/* Assign with := */}}
{{ $name := .Release.Name }}
{{ $image := printf "%s:%s" .Values.image.repository .Values.image.tag }}

{{/* Reassign with = (must be already declared) */}}
{{ $count := 0 }}
{{ range .Values.items }}
  {{ $count = add $count 1 }}
{{ end }}

{{/* Variables persist in scope — not inside range/with child scopes */}}
{{ $global := "hello" }}
{{ range .Values.items }}
  {{ $global }}     {{/* accessible — $-vars are visible in child scopes */}}
{{ end }}
```

### Pipelines — chaining functions

```gotemplate
{{/* Pipeline: output of left feeds as LAST arg to right */}}
{{ .Values.name | upper }}
{{ .Values.name | upper | trunc 63 | trimSuffix "-" }}

{{/* Equivalent forms */}}
{{ upper .Values.name }}
{{ .Values.name | upper }}

{{/* Multi-stage pipeline */}}
{{ .Values.config | toYaml | nindent 4 }}
{{ .Values.image.tag | default "latest" | quote }}

{{/* Pipeline in an if */}}
{{ if .Values.name | contains "prod" }}
  environment: production
{{ end }}
```

### Types

```gotemplate
{{/* Go templates handle these types */}}
string:  "hello"
int:     42
float:   3.14
bool:    true / false
nil:     nil
list:    []interface{}      (from YAML arrays)
map:     map[string]interface{}  (from YAML objects)
```

---

## 4. Control Structures

### `if` / `else if` / `else`

```gotemplate
{{ if .Values.enabled }}
enabled: true
{{ end }}

{{ if .Values.enabled }}
enabled: true
{{ else }}
enabled: false
{{ end }}

{{ if eq .Values.env "production" }}
replicas: 5
{{ else if eq .Values.env "staging" }}
replicas: 2
{{ else }}
replicas: 1
{{ end }}

{{/* Truthy values: non-zero, non-empty, non-nil, true */}}
{{/* Falsy values: 0, "", nil, false, empty list/map */}}
{{ if .Values.config }}     {{/* true if config is defined and non-empty */}}
{{ if not .Values.debug }}
{{ if and .Values.tls.enabled (not .Values.tls.selfSigned) }}
{{ if or .Values.expose.loadBalancer .Values.expose.ingress }}
```

### Comparison operators (must be functions, not operators)

```gotemplate
{{ if eq  .Values.env "production" }}    {{/* == */}}
{{ if ne  .Values.env "production" }}    {{/* != */}}
{{ if lt  .Values.replicas 3 }}          {{/* <  */}}
{{ if le  .Values.replicas 3 }}          {{/* <= */}}
{{ if gt  .Values.replicas 3 }}          {{/* >  */}}
{{ if ge  .Values.replicas 3 }}          {{/* >= */}}

{{/* Multiple conditions */}}
{{ if and (eq .Values.env "prod") (ge .Values.replicas 3) }}
{{ if or  (eq .Values.env "prod") (eq .Values.env "staging") }}
{{ if not (eq .Values.env "dev") }}
```

### `range` — iteration

```gotemplate
{{/* Range over a list */}}
{{ range .Values.hosts }}
- {{ . }}
{{ end }}

{{/* Range with index */}}
{{ range $i, $host := .Values.hosts }}
{{ $i }}: {{ $host }}
{{ end }}

{{/* Range over a map */}}
{{ range $key, $val := .Values.labels }}
{{ $key }}: {{ $val }}
{{ end }}

{{/* Range with else (empty list) */}}
{{ range .Values.items }}
  - {{ .name }}
{{ else }}
  # no items configured
{{ end }}

{{/* Range and build YAML list */}}
ports:
{{ range .Values.service.ports }}
  - name: {{ .name }}
    port: {{ .port }}
    targetPort: {{ .targetPort | default .port }}
{{ end }}
```

### `with` — narrow context

```gotemplate
{{/* with changes . to the value, skips block if falsy */}}
{{ with .Values.ingress }}
ingress:
  enabled: true
  host: {{ .host }}
  path: {{ .path | default "/" }}
{{ end }}

{{/* with + else */}}
{{ with .Values.resources }}
resources:
  {{- toYaml . | nindent 2 }}
{{ else }}
resources: {}
{{ end }}

{{/* Nested with — use $. to escape to root */}}
{{ with .Values.tls }}
  secretName: {{ $.Release.Name }}-tls
  host: {{ .host }}
{{ end }}
```

---

## 5. Functions & Sprig

### Built-in Go template functions

```gotemplate
{{/* Output / formatting */}}
{{ print "hello" " " "world" }}          {{/* concatenate: "hello world" */}}
{{ printf "%s:%d" .host .port }}         {{/* fmt.Sprintf */}}
{{ println "line" }}                     {{/* print with newline */}}

{{/* Logic */}}
{{ and true false }}                     {{/* false */}}
{{ or  false true }}                     {{/* true */}}
{{ not false }}                          {{/* true */}}

{{/* Comparison */}}
{{ eq  "a" "a" }}  {{ ne "a" "b" }}
{{ lt  1 2 }}      {{ le 1 1 }}
{{ gt  2 1 }}      {{ ge 2 2 }}

{{/* Type checking */}}
{{ len .Values.items }}                  {{/* length of list/map/string */}}
{{ index .Values.list 0 }}              {{/* list element by index */}}
{{ index .Values.map "key" }}           {{/* map lookup by key */}}
{{ call $func .Arg }}                   {{/* call a func value */}}
```

### Sprig functions (Helm) — string

```gotemplate
{{/* Case */}}
{{ "hello" | upper }}                   {{/* HELLO */}}
{{ "HELLO" | lower }}                   {{/* hello */}}
{{ "hello world" | title }}             {{/* Hello World */}}

{{/* Trim / pad */}}
{{ "  hello  " | trim }}               {{/* hello */}}
{{ "hello" | trimPrefix "hel" }}        {{/* lo */}}
{{ "hello" | trimSuffix "lo" }}         {{/* hel */}}
{{ "hello" | trunc 3 }}                {{/* hel */}}
{{ "hello" | trunc -3 }}               {{/* llo (from end) */}}
{{ "hello" | abbrev 4 }}               {{/* h... */}}
{{ "hello" | nospace }}                {{/* remove all whitespace */}}

{{/* Check / search */}}
{{ "hello" | contains "ell" }}          {{/* true */}}
{{ "hello" | hasPrefix "hel" }}         {{/* true */}}
{{ "hello" | hasSuffix "llo" }}         {{/* true */}}
{{ "hello" | regexMatch "^hel" }}       {{/* true */}}
{{ "hello world" | regexFind "\\w+" }}  {{/* hello */}}
{{ "hello world" | regexFindAll "\\w+" -1 }}  {{/* [hello world] */}}
{{ "hello" | regexReplaceAll "l" "r" }} {{/* herro */}}

{{/* Split / join */}}
{{ "a,b,c" | splitList "," }}           {{/* [a b c] */}}
{{ list "a" "b" "c" | join "," }}       {{/* a,b,c */}}
{{ "a.b.c" | split "." }}               {{/* map: $._0=a, $._1=b, $._2=c */}}
{{ "a.b.c" | splitn "." 2 }}            {{/* map: $._0=a, $._1=b.c */}}

{{/* Indent — critical for Helm YAML */}}
{{ .Values.config | toYaml | indent 4 }}    {{/* indent every line 4 spaces */}}
{{ .Values.config | toYaml | nindent 4 }}   {{/* newline + indent (most common) */}}

{{/* Quote / wrap */}}
{{ "hello" | quote }}                   {{/* "hello" */}}
{{ "hello" | squote }}                  {{/* 'hello' */}}
{{ list "a" "b" | join "," | quote }}   {{/* "a,b" */}}

{{/* Encode */}}
{{ "hello" | b64enc }}                  {{/* aGVsbG8= */}}
{{ "aGVsbG8=" | b64dec }}              {{/* hello */}}
{{ "hello world" | urlquery }}          {{/* hello+world */}}
{{ .Values.config | toJson }}
{{ .Values.config | toPrettyJson }}
{{ .Values.config | toYaml }}
```

### Sprig functions — math

```gotemplate
{{ add  1 2 }}    {{/* 3 */}}
{{ sub  5 3 }}    {{/* 2 */}}
{{ mul  3 4 }}    {{/* 12 */}}
{{ div  10 3 }}   {{/* 3 (integer division) */}}
{{ mod  10 3 }}   {{/* 1 */}}
{{ max  3 1 2 }}  {{/* 3 */}}
{{ min  3 1 2 }}  {{/* 1 */}}
{{ ceil  1.2 }}   {{/* 2 */}}
{{ floor 1.8 }}   {{/* 1 */}}
{{ round 3.567 2 }}  {{/* 3.57 */}}
{{ add1 5 }}         {{/* 6 — increment by 1 */}}
{{ int64 "42" }}     {{/* convert string to int64 */}}
{{ float64 "3.14" }} {{/* convert string to float64 */}}
```

### Sprig functions — lists

```gotemplate
{{ list "a" "b" "c" }}                  {{/* create list */}}
{{ list "a" "b" | append "c" }}         {{/* [a b c] */}}
{{ list "a" "b" "c" | prepend "z" }}    {{/* [z a b c] */}}
{{ list "a" "b" "c" | first }}          {{/* a */}}
{{ list "a" "b" "c" | last }}           {{/* c */}}
{{ list "a" "b" "c" | rest }}           {{/* [b c] */}}
{{ list "a" "b" "c" | initial }}        {{/* [a b] */}}
{{ list "a" "b" "a" | uniq }}           {{/* [a b] */}}
{{ list "c" "a" "b" | sortAlpha }}      {{/* [a b c] */}}
{{ list "a" "b" "c" | reverse }}        {{/* [c b a] */}}
{{ list "a" "b" "c" | without "b" }}    {{/* [a c] */}}
{{ list "a" "b" "c" | has "b" }}        {{/* true */}}
{{ concat (list "a") (list "b") }}      {{/* [a b] */}}
{{ list "a" "b" "c" | slice 0 2 }}      {{/* [a b] */}}
{{ list "a" "b" "c" | len }}            {{/* 3 */}}

{{/* chunk — split into sublists */}}
{{ list 1 2 3 4 5 | chunk 2 }}          {{/* [[1 2] [3 4] [5]] */}}
```

### Sprig functions — dicts

```gotemplate
{{ dict "key" "val" "key2" "val2" }}     {{/* create dict */}}
{{ $d := dict "a" 1 "b" 2 }}
{{ get $d "a" }}                         {{/* 1 */}}
{{ set $d "c" 3 }}                       {{/* mutates $d, adds key c */}}
{{ unset $d "a" }}                       {{/* mutates $d, removes key a */}}
{{ hasKey $d "b" }}                      {{/* true */}}
{{ keys $d | sortAlpha }}                {{/* sorted list of keys */}}
{{ values $d }}                          {{/* list of values */}}
{{ merge $d (dict "x" 10) }}             {{/* merge: $d wins on collision */}}
{{ mergeOverwrite $d (dict "a" 99) }}    {{/* merge: second wins */}}
{{ deepCopy $d }}                        {{/* deep clone */}}
{{ $d | toYaml }}                        {{/* serialize to YAML string */}}
{{ $d | toJson }}                        {{/* serialize to JSON string */}}
```

### Sprig functions — type conversion & checks

```gotemplate
{{ "42" | int }}           {{/* 42 */}}
{{ "3.14" | float64 }}     {{/* 3.14 */}}
{{ 42 | toString }}        {{/* "42" */}}
{{ "true" | toBool }}      {{/* true */}}

{{/* Type test */}}
{{ kindOf "hello" }}       {{/* string */}}
{{ kindOf 42 }}            {{/* int */}}
{{ kindOf .Values.list }}  {{/* slice */}}
{{ kindOf .Values.map }}   {{/* map */}}
{{ kindIs "string" .Val }} {{/* true/false */}}

{{/* deepEqual */}}
{{ deepEqual .Values.a .Values.b }}
```

### Sprig functions — dates

```gotemplate
{{ now | date "2006-01-02" }}                 {{/* current date: 2024-01-15 */}}
{{ now | date "2006-01-02T15:04:05Z" }}       {{/* ISO 8601 */}}
{{ now | unixEpoch }}                         {{/* unix timestamp as string */}}
{{ now | dateModify "+48h" | date "2006-01-02" }}   {{/* 2 days from now */}}
{{/* NOTE: Go date format is always the reference time: Mon Jan 2 15:04:05 MST 2006 */}}
```

### Sprig functions — crypto & random

```gotemplate
{{ randAlphaNum 16 }}      {{/* random 16-char alphanumeric */}}
{{ randAlpha 10 }}         {{/* random 10-char alpha only */}}
{{ randNumeric 8 }}        {{/* random 8-digit number string */}}
{{ randAscii 12 }}         {{/* random 12-char ASCII */}}

{{ "secret" | sha256sum }}
{{ "secret" | sha1sum }}
{{ "secret" | adler32sum }}

{{/* Generate self-signed cert (Helm secret pattern) */}}
{{ $ca := genCA "my-ca" 365 }}
{{ $cert := genSignedCert "my-svc" nil (list "my-svc" "my-svc.default") 365 $ca }}
tls.crt: {{ $cert.Cert | b64enc }}
tls.key: {{ $cert.Key  | b64enc }}
```

### Sprig — lookup & flow

```gotemplate
{{ default "fallback" .Values.optionalKey }}         {{/* value or fallback */}}
{{ .Values.key | default "fallback" }}               {{/* same, pipeline form */}}
{{ coalesce .Values.a .Values.b "fallback" }}        {{/* first non-empty value */}}
{{ empty .Values.key }}                              {{/* true if nil/zero/empty */}}
{{ required "msg" .Values.mustExist }}               {{/* fail with msg if empty */}}
{{ fail "custom error message" }}                    {{/* always fail with message */}}
{{ ternary "yes" "no" .Values.flag }}               {{/* if flag: "yes" else "no" */}}
```

---

## 6. Template Definition & Reuse

### Define and call named templates

```gotemplate
{{/* Define a named template (usually in _helpers.tpl) */}}
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/* Call a named template — pass . as context */}}
metadata:
  labels:
    {{- include "myapp.labels" . | nindent 4 }}

{{/* Call with modified context */}}
{{- include "myapp.labels" (dict "Release" .Release "Chart" .Chart "Values" .Values) }}

{{/* template vs include:
     template: outputs directly, can't be piped
     include:  returns string, can be piped to nindent etc. — always prefer include */}}
```

### Passing data to templates

```gotemplate
{{/* Pass a dict with multiple values */}}
{{- include "myapp.container" (dict "name" "sidecar" "image" .Values.sidecar.image "root" .) -}}

{{/* Inside the template, access via . */}}
{{- define "myapp.container" -}}
- name: {{ .name }}
  image: {{ .image }}
  {{- with .root.Values.resources }}
  resources:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

---

## 7. Helm — Chart Structure

```
my-chart/
├── Chart.yaml              # chart metadata
├── values.yaml             # default values
├── values.schema.json      # optional: JSON schema validation for values
├── .helmignore             # files to ignore (like .gitignore)
│
├── templates/
│   ├── _helpers.tpl        # named templates (not rendered directly — starts with _)
│   ├── NOTES.txt           # post-install notes (also templated)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   └── tests/
│       └── test-connection.yaml
│
├── charts/                 # subcharts (dependencies)
│   └── redis/
│
└── crds/                   # Custom Resource Definitions (applied before templates)
```

### `Chart.yaml`

```yaml
apiVersion: v2                  # v2 for Helm 3, v1 for Helm 2
name: my-chart
description: A Helm chart for my application
type: application               # application | library
version: 0.1.0                  # chart version (SemVer)
appVersion: "1.0.0"             # app version (informational)
keywords:
  - myapp
  - web
home: https://github.com/org/my-chart
sources:
  - https://github.com/org/myapp
maintainers:
  - name: Platform Team
    email: platform@example.com

dependencies:
  - name: redis
    version: "~17.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled          # only install if redis.enabled=true
  - name: postgresql
    version: "~12.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
    tags:
      - database
```

### `.helmignore`

```
# Patterns to ignore when packaging
.git/
.gitignore
*.tmp
tests/
README.md
```

---

## 8. Helm — Values & Overrides

### `values.yaml` — structure conventions

```yaml
# values.yaml — all defaults here

# Image
image:
  repository: myapp
  pullPolicy: IfNotPresent
  tag: ""                         # overridden by .Chart.AppVersion if empty

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

# Service account
serviceAccount:
  create: true
  annotations: {}
  name: ""                        # auto-generated if empty

# Pod settings
podAnnotations: {}
podLabels: {}
podSecurityContext: {}
securityContext: {}

# Service
service:
  type: ClusterIP
  port: 80

# Ingress
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

# Resources — leave empty to use cluster defaults
resources: {}
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

# Autoscaling
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

# Node placement
nodeSelector: {}
tolerations: []
affinity: {}

# Application-specific config
config:
  logLevel: info
  dbHost: localhost
  dbPort: 5432
  dbName: myapp

replicaCount: 1
```

### Value override hierarchy (lowest → highest)

```
1. chart/values.yaml           (defaults)
2. chart/values.yaml in parent (if subchart)
3. -f custom-values.yaml       (per -f flag, left to right)
4. --set key=value             (command line, highest)
5. --set-string key=value      (force string type)
6. --set-file key=filepath     (read value from file)
7. --set-json key=jsonvalue    (parse as JSON)
```

```bash
# Multiple -f files — later files win
helm upgrade myapp ./my-chart \
  -f values-base.yaml \
  -f values-production.yaml \
  --set image.tag=v1.2.3 \
  --set replicaCount=5

# Set nested values
helm install myapp ./my-chart \
  --set ingress.enabled=true \
  --set "ingress.hosts[0].host=myapp.example.com" \
  --set "ingress.hosts[0].paths[0].path=/"

# Set with special characters (use --set-string for strings with commas etc.)
helm install myapp ./my-chart \
  --set-string "podAnnotations.prometheus\.io/scrape=true"
```

---

## 9. Helm — Template Patterns

### Deployment template

```gotemplate
{{/* templates/deployment.yaml */}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "myapp.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          {{- with .Values.securityContext }}
          securityContext:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.env }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          envFrom:
            - configMapRef:
                name: {{ include "myapp.fullname" . }}-config
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Ingress template with API version detection

```gotemplate
{{/* templates/ingress.yaml */}}
{{- if .Values.ingress.enabled -}}
{{- $fullName := include "myapp.fullname" . -}}
{{- $svcPort := .Values.service.port -}}
{{- if and .Values.ingress.className (not (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion)) }}
  {{- if not (hasKey .Values.ingress.annotations "kubernetes.io/ingress.class") }}
  {{- $_ := set .Values.ingress.annotations "kubernetes.io/ingress.class" .Values.ingress.className}}
  {{- end }}
{{- end }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ $fullName }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if and .Values.ingress.className (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion) }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            {{- if semverCompare ">=1.18-0" $.Capabilities.KubeVersion.GitVersion }}
            pathType: {{ .pathType }}
            {{- end }}
            backend:
              {{- if semverCompare ">=1.19-0" $.Capabilities.KubeVersion.GitVersion }}
              service:
                name: {{ $fullName }}
                port:
                  number: {{ $svcPort }}
              {{- else }}
              serviceName: {{ $fullName }}
              servicePort: {{ $svcPort }}
              {{- end }}
          {{- end }}
    {{- end }}
{{- end }}
```

### ConfigMap template

```gotemplate
{{/* templates/configmap.yaml */}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
data:
  APP_LOG_LEVEL: {{ .Values.config.logLevel | quote }}
  APP_DB_HOST: {{ .Values.config.dbHost | quote }}
  APP_DB_PORT: {{ .Values.config.dbPort | toString | quote }}
  APP_DB_NAME: {{ .Values.config.dbName | quote }}
  {{- range $key, $val := .Values.config.extraEnv }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
  app.yaml: |
    {{- toYaml .Values.appConfig | nindent 4 }}
```

### Secret template

```gotemplate
{{/* templates/secret.yaml */}}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}-secret
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    {{/* Prevent Helm from diff-ing secret values */}}
    helm.sh/resource-policy: keep
type: Opaque
data:
  db-password: {{ .Values.config.dbPassword | b64enc | quote }}
  api-key: {{ .Values.config.apiKey | b64enc | quote }}
  {{/* Generate random password if not provided — note: regenerates on every upgrade! */}}
  jwt-secret: {{ .Values.config.jwtSecret | default (randAlphaNum 32) | b64enc | quote }}
```

---

## 10. Helm — Named Templates & Helpers

### `_helpers.tpl` — standard patterns

```gotemplate
{{/* templates/_helpers.tpl */}}

{{/*
Expand the name of the chart.
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
Truncate at 63 chars (Kubernetes name limit).
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart label value.
*/}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels — applied to all resources.
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels — used by Deployments and Services to match pods.
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Service account name.
*/}}
{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
Image reference — combine repository:tag.
*/}}
{{- define "myapp.image" -}}
{{- $tag := .Values.image.tag | default .Chart.AppVersion }}
{{- printf "%s:%s" .Values.image.repository $tag }}
{{- end }}

{{/*
Environment variables from config — reusable across containers.
*/}}
{{- define "myapp.envVars" -}}
- name: LOG_LEVEL
  value: {{ .Values.config.logLevel | quote }}
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: {{ include "myapp.fullname" . }}-config
      key: APP_DB_HOST
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: {{ include "myapp.fullname" . }}-secret
      key: db-password
{{- end }}
```

---

## 11. Helm — Flow Control & Conditionals

### Capabilities — version-aware templates

```gotemplate
{{/* Check Kubernetes version */}}
{{ if semverCompare ">=1.25-0" .Capabilities.KubeVersion.GitVersion }}
  {{/* use PodDisruptionBudget v1 */}}
{{ else }}
  {{/* use PodDisruptionBudget v1beta1 */}}
{{ end }}

{{/* Check if a CRD/API group exists */}}
{{ if .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
{{ if .Capabilities.APIVersions.Has "cert-manager.io/v1/Certificate" }}

{{/* Built-in objects in Helm templates */}}
.Release.Name          {{/* release name */}}
.Release.Namespace     {{/* namespace */}}
.Release.IsInstall     {{/* true on first install */}}
.Release.IsUpgrade     {{/* true on upgrade */}}
.Release.Service       {{/* "Helm" */}}
.Chart.Name            {{/* from Chart.yaml */}}
.Chart.Version         {{/* chart version */}}
.Chart.AppVersion      {{/* app version */}}
.Values                {{/* merged values */}}
.Files                 {{/* access to non-template files */}}
.Capabilities          {{/* cluster capabilities */}}
.Template.Name         {{/* current template filename */}}
.Template.BasePath     {{/* templates/ directory path */}}
```

### Accessing non-template files

```gotemplate
{{/* .Files — access files in the chart (not in templates/) */}}

{{/* Read a file as string */}}
{{ .Files.Get "configs/app.conf" }}

{{/* Embed file in ConfigMap */}}
apiVersion: v1
kind: ConfigMap
data:
  app.conf: |
    {{- .Files.Get "configs/app.conf" | nindent 4 }}

{{/* Read all files matching a glob */}}
{{- range $path, $content := .Files.Glob "configs/*.conf" }}
  {{ base $path }}: |
    {{- $content | toString | nindent 4 }}
{{- end }}

{{/* As base64 (for Secrets) */}}
data:
  app.conf: {{ .Files.Get "configs/app.conf" | b64enc }}

{{/* Files as a ConfigMap using AsConfig/AsSecrets */}}
data:
  {{- (.Files.Glob "configs/*").AsConfig | nindent 2 }}
```

### `lookup` — query live cluster state

```gotemplate
{{/* lookup "apiVersion" "kind" "namespace" "name" */}}
{{/* Returns empty dict if not found — safe to use in dry-run */}}

{{/* Check if a secret already exists (avoid overwriting) */}}
{{- $secret := lookup "v1" "Secret" .Release.Namespace (include "myapp.fullname" .) -}}
{{- if $secret }}
{{/* Secret exists — reuse existing value */}}
  mykey: {{ index $secret.data "mykey" }}
{{- else }}
{{/* Secret doesn't exist — generate */}}
  mykey: {{ randAlphaNum 32 | b64enc }}
{{- end }}

{{/* Check if namespace exists */}}
{{- $ns := lookup "v1" "Namespace" "" .Release.Namespace -}}
{{- if not $ns }}
  {{- fail (printf "namespace %s does not exist" .Release.Namespace) -}}
{{- end }}
```

---

## 12. Helm — Hooks & Tests

### Hooks

```gotemplate
{{/* templates/job-migrate.yaml — run DB migration before upgrade */}}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migrate
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"               {{/* lower = runs first */}}
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: {{ include "myapp.image" . }}
          command: ["myapp", "migrate"]
          env:
            {{- include "myapp.envVars" . | nindent 12 }}
```

```
Hook types:
  pre-install       → before any resources are created
  post-install      → after all resources are created
  pre-upgrade       → before upgrade
  post-upgrade      → after upgrade
  pre-rollback      → before rollback
  post-rollback     → after rollback
  pre-delete        → before delete
  post-delete       → after delete
  test              → run with helm test

Delete policies:
  before-hook-creation  → delete old hook resource before creating new
  hook-succeeded        → delete after successful completion
  hook-failed           → delete after failure
```

### Tests

```gotemplate
{{/* templates/tests/test-connection.yaml */}}
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "myapp.fullname" . }}-test-connection
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args:
        - '--spider'
        - 'http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}/health'
```

```bash
helm test myapp                      # run tests
helm test myapp --logs               # show test pod logs
```

---

## 13. kubectl — Output Formatting

### `-o go-template` for custom output

```bash
# Print a single field
kubectl get pod mypod -o go-template='{{ .status.phase }}'

# With newline
kubectl get pod mypod -o go-template='{{ .status.phase }}{{ "\n" }}'

# Iterate over items list
kubectl get pods -o go-template='
{{- range .items }}
{{ .metadata.name }} {{ .status.phase }}
{{- end }}'

# Filter: only Running pods
kubectl get pods -o go-template='
{{- range .items }}
{{- if eq .status.phase "Running" }}
{{ .metadata.name }}
{{- end }}
{{- end }}'

# Print specific container image
kubectl get pods -o go-template='
{{- range .items }}
{{ .metadata.name }}:
{{- range .spec.containers }}
  {{ .name }}: {{ .image }}
{{- end }}
{{- end }}'
```

### `-o go-template-file`

```gotemplate
{{/* pod-status.tmpl */}}
{{- range .items -}}
NAME: {{ .metadata.name }}
STATUS: {{ .status.phase }}
NODE: {{ .spec.nodeName }}
IP: {{ .status.podIP }}
AGE: {{ .metadata.creationTimestamp }}
---
{{ end -}}
```

```bash
kubectl get pods -o go-template-file=pod-status.tmpl
```

### JSONPath (simpler alternative for single fields)

```bash
# JSONPath — simpler for single-field extraction
kubectl get pod mypod -o jsonpath='{.status.phase}'
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# With filter
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# Multiple fields
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[-1].type}{"\n"}{end}'
```

### Useful one-liners

```bash
# List all images in cluster
kubectl get pods -A -o go-template='{{range .items}}{{range .spec.containers}}{{.image}}{{"\n"}}{{end}}{{end}}' | sort -u

# Get resource requests for all pods
kubectl get pods -A -o go-template='
{{- range .items }}
{{ .metadata.namespace }}/{{ .metadata.name }}:
{{- range .spec.containers }}
  {{ .name }}: cpu={{ if .resources.requests }}{{ .resources.requests.cpu }}{{ end }} mem={{ if .resources.requests }}{{ .resources.requests.memory }}{{ end }}
{{- end }}
{{ end }}'

# Show secret value (decoded)
kubectl get secret mysecret -o go-template='{{ index .data "password" | base64decode }}'

# Get all service ClusterIPs
kubectl get svc -A -o go-template='{{range .items}}{{.metadata.namespace}}/{{.metadata.name}}: {{.spec.clusterIP}}{{"\n"}}{{end}}'
```

---

## 14. Consul Template

### `.ctmpl` file basics

```gotemplate
{{/* consul-template reads from Consul KV, services, and Vault */}}

{{/* Read a Consul KV value */}}
{{ key "config/myapp/log-level" }}

{{/* KV with default */}}
{{ keyOrDefault "config/myapp/workers" "4" }}

{{/* List healthy instances of a service */}}
{{ range service "nginx" }}
server {{ .Address }}:{{ .Port }};
{{ end }}

{{/* Filter by tag */}}
{{ range service "app.production" }}
upstream {{ .Address }}:{{ .Port }};
{{ end }}

{{/* Read a Vault secret */}}
{{ with secret "secret/data/myapp/db" }}
DB_PASSWORD={{ .Data.data.password }}
{{ end }}

{{/* Dynamic nginx upstream config */}}
upstream myapp {
  least_conn;
  {{ range service "myapp" }}
  server {{ .Address }}:{{ .Port }} max_fails=3 fail_timeout=60s;
  {{ end }}
}
```

### Consul Template config (`config.hcl`)

```hcl
consul {
  address = "127.0.0.1:8500"
  token   = "{{ env "CONSUL_TOKEN" }}"
}

vault {
  address     = "https://vault.example.com:8200"
  token       = "{{ env "VAULT_TOKEN" }}"
  renew_token = true
}

template {
  source      = "/etc/consul-template/nginx.conf.ctmpl"
  destination = "/etc/nginx/nginx.conf"
  command     = "nginx -s reload"
  perms       = 0644
}
```

```bash
consul-template -config=config.hcl          # run in foreground
consul-template -config=config.hcl -once    # render once and exit
consul-template -template="src.ctmpl:dest.conf:nginx -s reload"
```

---

## 15. Common Gotchas & Best Practices

### Always use `include` over `template`

```gotemplate
{{/* WRONG — template cannot be piped */}}
{{ template "myapp.labels" . }}

{{/* CORRECT — include returns a string, can be piped to nindent */}}
{{ include "myapp.labels" . | nindent 4 }}
```

### `nindent` vs `indent`

```gotemplate
{{/* indent: adds spaces to each line — does NOT add leading newline */}}
labels:
  {{ include "myapp.labels" . | indent 2 }}   {{/* ← extra space before first line */}}

{{/* nindent: adds newline THEN indents — cleanest for YAML values */}}
labels:
  {{- include "myapp.labels" . | nindent 2 }}  {{/* ← trim before + newline + indent */}}

{{/* toYaml always use nindent pattern */}}
resources:
  {{- toYaml .Values.resources | nindent 2 }}
```

### YAML string quoting

```gotemplate
{{/* Always quote values that could be misinterpreted as YAML */}}
name: {{ .Values.name | quote }}           {{/* "my-app" */}}
tag:  {{ .Values.image.tag | quote }}      {{/* "1.0" → not interpreted as float */}}
port: {{ .Values.service.port }}           {{/* 80 — numbers are fine unquoted */}}

{{/* Danger: without quote, "1.0" becomes float 1 in YAML */}}
tag: {{ .Values.image.tag }}               {{/* BAD if tag is "1.0" */}}
tag: {{ .Values.image.tag | quote }}       {{/* GOOD: "1.0" */}}
```

### `required` — fail fast on missing values

```gotemplate
{{/* required "error message" .Value — fails at helm template/install if empty */}}
host: {{ required "ingress.host is required when ingress is enabled" .Values.ingress.host }}
password: {{ required "db.password must be set" .Values.db.password | b64enc }}
```

### Random values regenerate on every upgrade

```gotemplate
{{/* PROBLEM: randAlphaNum generates a new value on every helm upgrade */}}
{{/* This rotates secrets unexpectedly */}}
  jwt-secret: {{ randAlphaNum 32 | b64enc }}    {{/* ← BAD for stable secrets */}}

{{/* SOLUTION: Use lookup to reuse existing value */}}
{{- $existing := lookup "v1" "Secret" .Release.Namespace (include "myapp.fullname" .) -}}
{{- $secret := "" -}}
{{- if $existing -}}
  {{- $secret = index $existing.data "jwt-secret" | b64dec -}}
{{- else -}}
  {{- $secret = randAlphaNum 32 -}}
{{- end }}
  jwt-secret: {{ $secret | b64enc }}
```

### Helm lint and dry-run

```bash
# Lint — catches syntax errors and best-practice violations
helm lint ./my-chart
helm lint ./my-chart -f values-production.yaml

# Render templates locally (no cluster needed)
helm template myapp ./my-chart
helm template myapp ./my-chart -f values-production.yaml

# Dry-run against cluster (server-side validation)
helm install myapp ./my-chart --dry-run
helm upgrade myapp ./my-chart --dry-run

# Debug — render with extra output
helm install myapp ./my-chart --dry-run --debug

# Diff (requires helm-diff plugin)
helm diff upgrade myapp ./my-chart -f values-production.yaml
```

### Helm CLI workflow

```bash
# Add a chart repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search
helm search repo nginx
helm search hub nginx                        # search Artifact Hub

# Install
helm install myapp ./my-chart
helm install myapp ./my-chart -n mynamespace --create-namespace
helm install myapp bitnami/nginx --version 15.0.0

# Upgrade (install if not exists)
helm upgrade --install myapp ./my-chart -f values-prod.yaml

# List releases
helm list -A                                # all namespaces
helm list -n mynamespace

# Status
helm status myapp
helm get values myapp                       # values used in last deploy
helm get manifest myapp                     # rendered manifests
helm get notes myapp

# Rollback
helm history myapp                          # list revisions
helm rollback myapp 2                       # rollback to revision 2

# Uninstall
helm uninstall myapp
helm uninstall myapp --keep-history         # keep history for rollback

# Package a chart
helm package ./my-chart
helm package ./my-chart --version 0.2.0

# Manage dependencies
helm dependency update ./my-chart           # download charts/ dependencies
helm dependency list ./my-chart
```

---

## 16. Quick Reference Cheat Sheet

```gotemplate
{{/* ─── DELIMITERS ──────────────────────────────────────────────────── */}}
{{ .Value }}                        output
{{- .Value }}                       output, trim whitespace before
{{ .Value -}}                       output, trim whitespace after
{{- .Value -}}                      output, trim both sides
{{/* comment */}}                   comment — stripped from output

{{/* ─── CONTEXT (DOT) ──────────────────────────────────────────────── */}}
.                                   current context
.Release.Name / .Release.Namespace
.Chart.Name / .Chart.AppVersion
.Values.key.nested
$.Values.key                        root context from inside range/with
$var := .Value                      variable assignment

{{/* ─── CONTROL FLOW ────────────────────────────────────────────────── */}}
{{ if COND }} ... {{ else if }} ... {{ else }} ... {{ end }}
{{ range .List }} {{ . }} {{ end }}
{{ range $i, $v := .List }} {{ $i }}: {{ $v }} {{ end }}
{{ range $k, $v := .Map }} {{ $k }}: {{ $v }} {{ end }}
{{ with .Value }} {{ . }} {{ end }}   (skip block if falsy, . = value)

{{/* ─── COMPARISON (functions, not operators) ────────────────────────── */}}
eq ne lt le gt ge and or not

{{/* ─── PIPELINES ──────────────────────────────────────────────────── */}}
{{ .Val | upper | trunc 63 | trimSuffix "-" }}
{{ .Val | default "fallback" | quote }}
{{ .Val | toYaml | nindent 4 }}

{{/* ─── NAMED TEMPLATES ──────────────────────────────────────────────── */}}
{{- define "chart.helper" -}} ... {{- end }}
{{- include "chart.helper" . | nindent 4 }}     ← always include over template
{{- include "chart.helper" (dict "key" "val" "root" .) }}

{{/* ─── SPRIG STRINGS ────────────────────────────────────────────────── */}}
upper lower title trim trimPrefix trimSuffix
trunc abbrev nospace replace
contains hasPrefix hasSuffix
regexMatch regexFind regexFindAll regexReplaceAll
splitList join quote squote
b64enc b64dec urlquery
indent nindent
toYaml toJson toPrettyJson

{{/* ─── SPRIG LISTS ────────────────────────────────────────────────── */}}
list append prepend first last rest initial
uniq sortAlpha reverse without has concat slice len chunk

{{/* ─── SPRIG DICTS ────────────────────────────────────────────────── */}}
dict get set unset hasKey keys values
merge mergeOverwrite deepCopy toYaml toJson

{{/* ─── SPRIG FLOW ────────────────────────────────────────────────── */}}
default   coalesce   required   fail
empty     ternary
kindOf    kindIs     deepEqual

{{/* ─── SPRIG MATH ────────────────────────────────────────────────── */}}
add sub mul div mod max min ceil floor round add1

{{/* ─── SPRIG CRYPTO ──────────────────────────────────────────────── */}}
randAlphaNum randAlpha randNumeric sha256sum sha1sum

{{/* ─── HELM BUILT-INS ───────────────────────────────────────────── */}}
semverCompare ">=1.25-0" .Capabilities.KubeVersion.GitVersion
.Capabilities.APIVersions.Has "networking.k8s.io/v1"
.Files.Get "path"      .Files.Glob "configs/*"
lookup "v1" "Secret" .Release.Namespace "name"
```

### Helm CLI workflow summary

```bash
helm repo add <name> <url>        helm repo update
helm search repo <keyword>
helm install <rel> <chart>        -n <ns> --create-namespace -f vals.yaml
helm upgrade --install <rel> <chart>  -f vals.yaml --set key=val
helm template <rel> <chart>       # render locally (no cluster)
helm lint <chart>                 # syntax + best-practices check
helm diff upgrade <rel> <chart>   # requires helm-diff plugin
helm list -A                      helm status <rel>
helm get values <rel>             helm get manifest <rel>
helm rollback <rel> <revision>    helm history <rel>
helm uninstall <rel>
helm dependency update <chart>
helm package <chart>
helm test <rel>
```

### Key rules at a glance

| Rule | Detail |
|------|--------|
| `include` over `template` | `template` can't be piped; `include` returns a string |
| `nindent` over `indent` | `nindent` adds leading newline + indent — cleaner YAML |
| Quote string values | `\| quote` prevents YAML misinterpreting `"1.0"` as float `1` |
| `required` for mandatory values | Fail at template time with a clear message, not at runtime |
| `{{- -}}` trim whitespace | Prevent blank lines and leading spaces in rendered YAML |
| `$.` for root in range/with | `.` changes context — use `$.Values` to access root |
| `lookup` for stable secrets | `randAlphaNum` regenerates every upgrade — use `lookup` to reuse |
| `semverCompare` for API versions | Detect cluster version to use correct apiVersion |
| `helm template` first | Render locally before hitting the cluster |
| `_helpers.tpl` for shared logic | All reusable named templates go in `_helpers.tpl` |
| Truncate names to 63 chars | Kubernetes name limit: `trunc 63 \| trimSuffix "-"` |
| `helm diff` before upgrade | Never upgrade production without reviewing the diff |

---

*Part of DevOpsNotes / LANGUAGES — see also `03_HCL.md`, `04_Jinja2.md`, `06_TOML_XML.md`*