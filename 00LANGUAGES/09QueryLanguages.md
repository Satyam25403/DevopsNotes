# Makefile — DevOps Reference Notes

> **Make** — a build automation tool from 1976 that remains ubiquitous in DevOps. Makefiles define **targets**, **dependencies**, and **recipes** — giving every project a consistent, self-documenting task runner interface regardless of the underlying tech stack. Used for: building Go/C/C++/Rust binaries, orchestrating Docker builds, wrapping Terraform/Helm/kubectl workflows, running test suites, managing local dev environments, and providing `make help` as a project's single entry point. GNU Make is the standard on Linux; `gmake` on macOS (Homebrew).

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Syntax Fundamentals](#2-syntax-fundamentals)
3. [Variables](#3-variables)
4. [Automatic Variables](#4-automatic-variables)
5. [Pattern Rules & Static Pattern Rules](#5-pattern-rules--static-pattern-rules)
6. [Functions](#6-functions)
7. [Conditionals](#7-conditionals)
8. [Includes & Modularity](#8-includes--modularity)
9. [Phony Targets & Special Targets](#9-phony-targets--special-targets)
10. [Makefile as Task Runner — Common Patterns](#10-makefile-as-task-runner--common-patterns)
11. [Go Project Makefile](#11-go-project-makefile)
12. [Docker & Container Workflows](#12-docker--container-workflows)
13. [Terraform / Infrastructure Makefile](#13-terraform--infrastructure-makefile)
14. [Kubernetes & Helm Makefile](#14-kubernetes--helm-makefile)
15. [Common Gotchas & Best Practices](#15-common-gotchas--best-practices)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Core Concepts

```
A Makefile describes a dependency graph:
  target: dependency1 dependency2
  [TAB]   recipe_command
  [TAB]   another_command

Make runs the recipe only if:
  - The target file doesn't exist, OR
  - A dependency file is newer than the target

For .PHONY targets (tasks, not files):
  - Recipe always runs regardless of file state
```

```
How Make executes:
  make              → runs first (default) target
  make build        → runs the 'build' target
  make test lint    → runs 'test' then 'lint'
  make -n build     → dry-run: print commands without executing
  make -B build     → force rebuild (ignore timestamps)
  make -j4          → run 4 targets in parallel
  make -C ./subdir  → run make in a subdirectory
  make -f other.mk  → use a different Makefile
  make VERBOSE=1    → pass variable override
```

Critical rule: **recipes must be indented with a TAB character, never spaces.** This is the #1 source of cryptic Makefile errors.

---

## 2. Syntax Fundamentals

### Basic rule structure

```makefile
# target: prerequisites
#[TAB]recipe line 1
#[TAB]recipe line 2

build: main.go go.mod
	go build -o bin/app ./...

# Multiple prerequisites
test: build fixtures/data.json
	go test ./...

# No prerequisites — always runs when explicitly called
clean:
	rm -rf bin/ dist/
```

### Comments

```makefile
# This is a comment
target: dep   # inline comment on rule line is fine
	# this is a comment in a recipe — it gets echoed unless prefixed with @
	@# this recipe comment is NOT echoed (@ suppresses output)
	command
```

### Recipe prefixes

```makefile
target:
	command          # echoed to stdout, then executed
	@command         # @ suppresses echoing — run silently
	-command         # - ignores non-zero exit code (don't fail on error)
	@-command        # both: silent + ignore errors
	+command         # run even with --dry-run / -n flag

# Example
deploy:
	@echo "Deploying to production..."
	@kubectl apply -f manifests/
	@-kubectl rollout status deployment/myapp --timeout=120s
```

### Multi-line recipes

```makefile
# Each line in a recipe is a separate shell invocation
# Variables set in one line are NOT visible in the next line!

# WRONG — $VAR not visible in second line
bad-example:
	VAR=hello
	echo $$VAR          # empty — separate shell

# CORRECT — use \ to join lines into one shell invocation
good-example:
	VAR=hello; \
	echo $$VAR

# Or use .ONESHELL (GNU Make 3.82+)
.ONESHELL:
oneshell-example:
	VAR=hello
	echo $$VAR          # works — same shell session
```

### Shell in recipes

```makefile
# Default shell is /bin/sh — use SHELL to override
SHELL := /bin/bash

# Enable bash strict mode in all recipes
.SHELLFLAGS := -eu -o pipefail -c

# Dollar signs: $$ in recipe = literal $ passed to shell
#               $VAR         = Make variable expansion
print-pwd:
	@echo "Make dir: $(CURDIR)"
	@echo "Shell pid: $$$$"          # $$$$ → $$ in shell → $$
	@for i in 1 2 3; do echo $$i; done
```

---

## 3. Variables

### Variable assignment flavors

```makefile
# = (recursive) — expanded every time variable is used
CC = gcc
CFLAGS = $(EXTRA) -Wall       # EXTRA resolved at use time

# := (simply expanded) — expanded once at assignment time
VERSION := $(shell git describe --tags --always --dirty)
BUILD_TIME := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)

# ?= (conditional) — set only if not already set
ENVIRONMENT ?= development
IMAGE_TAG   ?= latest
REGISTRY    ?= docker.io/myorg

# += (append)
CFLAGS := -Wall
CFLAGS += -Wextra -O2         # CFLAGS = -Wall -Wextra -O2

# != (shell assignment, GNU Make 4.0+)
GIT_COMMIT != git rev-parse --short HEAD
# Equivalent to:
GIT_COMMIT := $(shell git rev-parse --short HEAD)
```

### Variable references

```makefile
$(VAR)          # standard reference — always use this form
${VAR}          # equivalent, less common
$V              # single-char variable — avoid (confusing)

# Nested expansion
ENV   := prod
IMAGE := myapp-$(ENV)        # myapp-prod
```

### Environment variables

```makefile
# Environment variables are available as Make variables
# Make variables override environment variables by default
# Use 'override' to prevent command-line override:

override VERSION := 1.0.0    # cannot be overridden from CLI

# Export Make variable to child processes / recipes
export DATABASE_URL := postgres://localhost/dev
export GOPATH       := $(HOME)/go

# Export all variables to child shells
.EXPORT_ALL_VARIABLES:
```

### Command-line variable overrides

```bash
# Override any variable at the command line
make build IMAGE_TAG=v1.2.3
make deploy ENVIRONMENT=production REGISTRY=my.registry.io
make test VERBOSE=1

# In Makefile: use ?= for overridable defaults
IMAGE_TAG   ?= $(shell git rev-parse --short HEAD)
ENVIRONMENT ?= development
```

### Computed variables

```makefile
# Git-derived values — best with :=
GIT_COMMIT    := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
GIT_TAG       := $(shell git describe --tags --exact-match 2>/dev/null || echo "")
GIT_BRANCH    := $(shell git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "unknown")
GIT_DIRTY     := $(shell git status --porcelain 2>/dev/null | wc -l | tr -d ' ')
BUILD_TIME    := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)
BUILD_HOST    := $(shell hostname)

# Detect OS
OS            := $(shell uname -s | tr '[:upper:]' '[:lower:]')
ARCH          := $(shell uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')

# Project layout
ROOT_DIR      := $(shell git rev-parse --show-toplevel 2>/dev/null || pwd)
BIN_DIR       := $(ROOT_DIR)/bin
DIST_DIR      := $(ROOT_DIR)/dist

# Conditional based on OS
ifeq ($(OS),darwin)
  SED := gsed       # GNU sed on macOS (brew install gnu-sed)
else
  SED := sed
endif
```

---

## 4. Automatic Variables

Automatic variables are set by Make for each rule and refer to parts of the rule itself.

```makefile
$@    # target name
$<    # first prerequisite
$^    # all prerequisites (deduplicated, space-separated)
$+    # all prerequisites (with duplicates)
$?    # prerequisites newer than the target
$*    # stem — matched part in a pattern rule (% match)
$(@D) # directory part of target
$(@F) # file part of target
$(<D) # directory part of first prerequisite
$(<F) # file part of first prerequisite

# Examples
bin/app: cmd/main.go pkg/server.go
	@echo "Target:        $@"      # bin/app
	@echo "First dep:     $<"      # cmd/main.go
	@echo "All deps:      $^"      # cmd/main.go pkg/server.go
	@echo "Target dir:    $(@D)"   # bin
	@echo "Target file:   $(@F)"   # app
	go build -o $@ ./cmd/

# Pattern rule using $*
%.png: %.svg
	inkscape --export-png=$@ $<
	@echo "Converted $* from SVG to PNG"

# Building multiple binaries
BINARIES := api worker cli

$(BINARIES): %: cmd/%/main.go
	@mkdir -p bin/
	go build -o bin/$@ ./cmd/$@/
```

---

## 5. Pattern Rules & Static Pattern Rules

### Pattern rules

```makefile
# % is the wildcard stem

# Compile all .c files to .o files
%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

# All .proto files → generated Go files
%.pb.go: %.proto
	protoc --go_out=. --go-grpc_out=. $<

# Build any binary from its cmd/ directory
bin/%: cmd/%/main.go
	@mkdir -p $(@D)
	go build -ldflags "$(LDFLAGS)" -o $@ ./cmd/$*/

# Minify JS files
dist/%.min.js: src/%.js
	terser $< -o $@ --compress --mangle
```

### Static pattern rules — apply pattern to a specific list

```makefile
OBJECTS := main.o server.o handler.o

# Apply %.o: %.c rule only to OBJECTS list
$(OBJECTS): %.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

# Build specific binaries from their directories
SERVICES := api worker scheduler

$(addprefix bin/, $(SERVICES)): bin/%: cmd/%/main.go
	go build -o $@ ./cmd/$*/
```

---

## 6. Functions

### String functions

```makefile
# $(subst from,to,text)
FILES := a.c b.c c.c
OBJS  := $(subst .c,.o,$(FILES))     # a.o b.o c.o

# $(patsubst pattern,replacement,text)  — like subst but with %
SRCS  := src/a.go src/b.go src/c.go
PKGS  := $(patsubst src/%.go,%,$(SRCS))   # a b c
# Common shorthand:
PKGS  := $(SRCS:src/%.go=%)               # same result

# $(strip text) — remove leading/trailing whitespace
MSG   := $(strip   hello world   )        # "hello world"

# $(findstring find,in)
ifneq ($(findstring arm64,$(ARCH)),)
  GOARCH := arm64
endif

# $(filter pattern...,text) — keep matching words
GOFILES := $(filter %.go, $(wildcard src/*))

# $(filter-out pattern...,text) — remove matching words
ALL_FILES := main.go main_test.go util.go util_test.go
SRC_ONLY  := $(filter-out %_test.go, $(ALL_FILES))

# $(sort list) — sort + deduplicate
SORTED := $(sort c b a b c)              # a b c

# $(word n,text) — nth word (1-based)
FIRST := $(word 1, one two three)        # one

# $(words text) — word count
COUNT := $(words one two three)          # 3

# $(wordlist s,e,text) — words s through e
SLICE := $(wordlist 2,3, one two three four)  # two three

# $(firstword text)
# $(lastword text)
FIRST := $(firstword one two three)      # one
LAST  := $(lastword  one two three)      # three

# $(join list1,list2) — pairwise concatenation
RESULT := $(join a b c, 1 2 3)           # a1 b2 c3
```

### File functions

```makefile
# $(wildcard pattern) — glob expansion
GO_FILES := $(wildcard **/*.go)
TF_FILES := $(wildcard terraform/**/*.tf)

# $(dir path) — directory component (with trailing /)
DIR := $(dir src/pkg/main.go)            # src/pkg/

# $(notdir path) — filename without directory
FILE := $(notdir src/pkg/main.go)        # main.go

# $(basename path) — remove extension
BASE := $(basename src/main.go)          # src/main

# $(suffix path) — extension only
EXT := $(suffix main.go)                 # .go

# $(addsuffix suffix,list)
BINS := $(addsuffix .exe, app server)    # app.exe server.exe

# $(addprefix prefix,list)
PATHS := $(addprefix bin/, app server)   # bin/app bin/server

# $(abspath path) — resolve to absolute path
ABS := $(abspath ../other/file.go)

# $(realpath path) — like abspath but resolves symlinks
```

### Control flow functions

```makefile
# $(if condition,then[,else])
MODE := $(if $(DEBUG),debug,release)

# $(or val1,val2,val3) — first non-empty value
TAG := $(or $(GIT_TAG),$(GIT_COMMIT),latest)

# $(and val1,val2) — empty if any arg is empty, else last arg
BOTH := $(and $(A),$(B))

# $(foreach var,list,text) — loop
CLEAN_DIRS := bin dist .cache
clean:
	$(foreach d, $(CLEAN_DIRS), rm -rf $(d);)
	# expands to: rm -rf bin; rm -rf dist; rm -rf .cache;

# Build a list with foreach
IMAGES := app worker cli
TAGS   := $(foreach img, $(IMAGES), $(REGISTRY)/$(img):$(VERSION))

# $(call var,param1,param2) — call a named function
define docker-build
	docker build \
		--file $(1)/Dockerfile \
		--tag $(REGISTRY)/$(1):$(VERSION) \
		$(1)/
endef

build-app:    ; $(call docker-build,app)
build-worker: ; $(call docker-build,worker)

# $(eval text) — evaluate text as Makefile syntax
define make-target
$(1):
	@echo "Running $(1)"
	$(2)
endef

$(eval $(call make-target,greet,echo "Hello"))

# $(value var) — get unexpanded value of recursive variable
# $(origin var) — where variable was defined: default, environment, file, command line
# $(flavor var) — variable type: recursive, simple, undefined
```

### Shell function

```makefile
# $(shell command) — run command, capture output
# IMPORTANT: use := not = to avoid re-running on every reference

VERSION    := $(shell git describe --tags --always 2>/dev/null)
GO_VERSION := $(shell go version | cut -d' ' -f3)
DOCKER_OK  := $(shell docker info >/dev/null 2>&1 && echo yes || echo no)

# Multi-line shell output — newlines become spaces
GO_PKGS := $(shell go list ./...)

# Check if a command exists
HAS_JQ := $(shell command -v jq 2>/dev/null)
```

---

## 7. Conditionals

### `ifeq` / `ifneq` / `ifdef` / `ifndef`

```makefile
# ifeq (a,b) — equal
ifeq ($(ENVIRONMENT),production)
  REPLICAS  := 5
  LOG_LEVEL := warn
else ifeq ($(ENVIRONMENT),staging)
  REPLICAS  := 2
  LOG_LEVEL := info
else
  REPLICAS  := 1
  LOG_LEVEL := debug
endif

# ifneq (a,b) — not equal
ifneq ($(GIT_DIRTY),0)
  VERSION_SUFFIX := -dirty
endif

# ifdef / ifndef — variable defined or not
ifdef CI
  DOCKER_FLAGS += --no-cache
  TEST_FLAGS   += -count=1
endif

ifndef REGISTRY
  $(error REGISTRY is not set. Usage: make push REGISTRY=my.registry.io)
endif

# Inline conditional variable
DOCKER_FLAGS := $(if $(CI),--no-cache,) --progress=auto
```

### Guard targets — validate required variables

```makefile
# Reusable guard — fails with clear message if variable not set
guard-%:
	@test -n "$($*)" || (echo "ERROR: $* is not set"; exit 1)

# Use as prerequisite
deploy: guard-ENVIRONMENT guard-IMAGE_TAG
	kubectl set image deployment/myapp app=$(REGISTRY)/myapp:$(IMAGE_TAG)

push: guard-REGISTRY
	docker push $(REGISTRY)/myapp:$(VERSION)

# Example usage:
# make deploy ENVIRONMENT=production IMAGE_TAG=v1.2.3
# make deploy  ← fails with "ERROR: ENVIRONMENT is not set"
```

---

## 8. Includes & Modularity

```makefile
# include reads another Makefile — paths relative to current file
include common.mk
include $(ROOT_DIR)/build/rules.mk

# -include (or sinclude) — don't error if file missing
-include .env.mk           # optional local overrides
-include $(HOME)/.config/make/defaults.mk

# include generated dependency files
-include $(OBJECTS:.o=.d)  # auto-generated .d files from gcc -MMD

# Split a large Makefile into sections
include build/variables.mk
include build/docker.mk
include build/k8s.mk
include build/terraform.mk
```

### `.env` file loading pattern

```makefile
# Load .env file into Make variables
# .env format: KEY=VALUE (no export, no quotes)
-include .env
export $(shell sed 's/=.*//' .env 2>/dev/null)

# Or more robust:
ifneq (,$(wildcard .env))
  include .env
  export
endif
```

---

## 9. Phony Targets & Special Targets

### `.PHONY` — targets that aren't files

```makefile
# Declare all task targets as PHONY
# Without this, Make skips the recipe if a file with that name exists
.PHONY: all build test lint clean docker-build deploy help

# Good practice: declare .PHONY for everything that isn't a real file target
.PHONY: \
	build test lint fmt vet \
	docker-build docker-push docker-run \
	deploy rollback \
	tf-init tf-plan tf-apply \
	helm-lint helm-deploy \
	clean clean-docker \
	help
```

### Special built-in targets

```makefile
# .DEFAULT_GOAL — set the default target (instead of first target)
.DEFAULT_GOAL := help

# .ONESHELL — run all recipe lines in a single shell
.ONESHELL:

# .SHELLFLAGS — flags passed to SHELL
SHELL        := /bin/bash
.SHELLFLAGS  := -eu -o pipefail -c

# .DELETE_ON_ERROR — delete target file if recipe fails
.DELETE_ON_ERROR:

# .SILENT — suppress recipe echoing for listed targets
.SILENT: clean

# .SUFFIXES — clear default suffix rules (legacy)
.SUFFIXES:

# .SECONDEXPANSION — enable $$ in prerequisites for deferred expansion
.SECONDEXPANSION:

# .NOTPARALLEL — prevent parallel execution for listed targets
.NOTPARALLEL: deploy rollback
```

### Self-documenting `help` target

```makefile
.DEFAULT_GOAL := help

## help: Show this help message
help:
	@echo "Usage: make [target]"
	@echo ""
	@grep -E '^## ' $(MAKEFILE_LIST) \
		| sed 's/## //' \
		| awk 'BEGIN{FS=":"} {printf "  \033[36m%-25s\033[0m %s\n", $$1, $$2}'

## build: Compile the application
build:
	go build -o bin/app ./...

## test: Run all tests
test:
	go test ./...

## lint: Run linters
lint:
	golangci-lint run

# ── Alternative: two-hash comment style ────────────────────────────────────
.PHONY: help
help: ## Display this help
	@awk 'BEGIN {FS = ":.*##"; printf "Usage:\n  make \033[36m<target>\033[0m\n\nTargets:\n"} \
	/^[a-zA-Z_0-9-]+:.*?##/ { printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2 }' \
	$(MAKEFILE_LIST)

build: ## Build the application binary
	go build -o bin/app ./...

test: ## Run unit tests
	go test ./...

lint: ## Run linters (golangci-lint)
	golangci-lint run
```

---

## 10. Makefile as Task Runner — Common Patterns

### Default all target

```makefile
.DEFAULT_GOAL := help

.PHONY: all
all: lint test build  ## Run lint, test, then build

.PHONY: ci
ci: deps lint test build docker-build  ## Full CI pipeline
```

### Versioning and build info

```makefile
# Version from git
VERSION       := $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
GIT_COMMIT    := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
GIT_BRANCH    := $(shell git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "unknown")
BUILD_TIME    := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)

# Inject into Go binary via ldflags
LDFLAGS := -ldflags "\
  -X main.Version=$(VERSION) \
  -X main.GitCommit=$(GIT_COMMIT) \
  -X main.BuildTime=$(BUILD_TIME) \
  -w -s"

.PHONY: build
build: ## Build the binary with version info
	@mkdir -p bin/
	go build $(LDFLAGS) -o bin/$(APP_NAME) ./cmd/$(APP_NAME)/

.PHONY: version
version: ## Print version info
	@echo "Version:    $(VERSION)"
	@echo "Git commit: $(GIT_COMMIT)"
	@echo "Git branch: $(GIT_BRANCH)"
	@echo "Build time: $(BUILD_TIME)"
```

### Dependency checking

```makefile
# Check required tools are installed
REQUIRED_TOOLS := docker kubectl helm terraform jq
check-tools: ## Check required tools are installed
	@for tool in $(REQUIRED_TOOLS); do \
		if ! command -v $$tool &>/dev/null; then \
			echo "ERROR: $$tool is not installed"; \
			exit 1; \
		else \
			echo "  ✓ $$tool ($$($$tool --version 2>&1 | head -1))"; \
		fi; \
	done

# Per-tool checks as prerequisites
.check-docker:
	@command -v docker &>/dev/null || (echo "docker is required but not installed"; exit 1)
	@docker info &>/dev/null || (echo "Docker daemon is not running"; exit 1)

docker-build: .check-docker
	docker build -t $(IMAGE) .
```

### Directory creation

```makefile
DIRS := bin dist .cache tmp/logs

# Create all dirs as a dependency
$(DIRS):
	@mkdir -p $@

build: | bin   # order-only prerequisite — create bin/ if needed, don't rebuild on change
	go build -o bin/app ./...

# Or create inline
bin/app: cmd/main.go
	@mkdir -p $(@D)
	go build -o $@ ./cmd/
```

### Cleanup targets

```makefile
.PHONY: clean clean-all clean-docker

clean: ## Remove build artifacts
	@rm -rf bin/ dist/ coverage.out coverage.html
	@find . -name '*.test' -delete
	@find . -name '*.out' -delete
	@echo "  cleaned build artifacts"

clean-docker: ## Remove project Docker images
	@docker images --filter "reference=$(REGISTRY)/$(APP_NAME)*" -q \
		| xargs -r docker rmi -f
	@echo "  cleaned Docker images"

clean-all: clean clean-docker ## Remove everything including caches
	@go clean -cache -testcache -modcache
	@echo "  cleaned all caches"
```

---

## 11. Go Project Makefile

```makefile
# ============================================================================
# Go project Makefile
# ============================================================================

# ── Project settings ─────────────────────────────────────────────────────────
APP_NAME    := myapp
MODULE      := github.com/org/$(APP_NAME)
REGISTRY    ?= docker.io/myorg
IMAGE       := $(REGISTRY)/$(APP_NAME)

# ── Versioning ───────────────────────────────────────────────────────────────
VERSION     := $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
GIT_COMMIT  := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
BUILD_TIME  := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)
LDFLAGS     := -ldflags "-X $(MODULE)/internal/version.Version=$(VERSION) \
                          -X $(MODULE)/internal/version.GitCommit=$(GIT_COMMIT) \
                          -X $(MODULE)/internal/version.BuildTime=$(BUILD_TIME) \
                          -w -s"

# ── Go settings ──────────────────────────────────────────────────────────────
SHELL       := /bin/bash
.SHELLFLAGS := -eu -o pipefail -c

GO          := go
GOFLAGS     :=
GOOS        ?= $(shell go env GOOS)
GOARCH      ?= $(shell go env GOARCH)
CGO_ENABLED ?= 0

GO_PKGS     := $(shell go list ./...)
GO_FILES    := $(shell find . -name '*.go' -not -path './vendor/*')

# ── Directories ──────────────────────────────────────────────────────────────
BIN_DIR     := bin
DIST_DIR    := dist
COVER_DIR   := .coverage

# ── Tools ────────────────────────────────────────────────────────────────────
GOLANGCI_LINT_VERSION := v1.55.2
GOLANGCI_LINT         := $(BIN_DIR)/golangci-lint

.DEFAULT_GOAL := help
.PHONY: all build build-all test test-race test-cover lint fmt vet \
        clean docker-build docker-push run generate tidy check deps help

# ── Default ──────────────────────────────────────────────────────────────────
all: lint test build ## Run lint, test, build

# ── Build ─────────────────────────────────────────────────────────────────────
build: ## Build binary for current OS/ARCH
	@mkdir -p $(BIN_DIR)
	CGO_ENABLED=$(CGO_ENABLED) GOOS=$(GOOS) GOARCH=$(GOARCH) \
		$(GO) build $(LDFLAGS) -o $(BIN_DIR)/$(APP_NAME) ./cmd/$(APP_NAME)/
	@echo "  built $(BIN_DIR)/$(APP_NAME) ($(GOOS)/$(GOARCH))"

build-linux: ## Build binary for linux/amd64
	GOOS=linux GOARCH=amd64 $(MAKE) build

build-all: ## Build for linux/amd64, linux/arm64, darwin/amd64, darwin/arm64
	@mkdir -p $(DIST_DIR)
	@for os in linux darwin; do \
		for arch in amd64 arm64; do \
			echo "  building $$os/$$arch..."; \
			CGO_ENABLED=0 GOOS=$$os GOARCH=$$arch \
				$(GO) build $(LDFLAGS) \
				-o $(DIST_DIR)/$(APP_NAME)-$$os-$$arch \
				./cmd/$(APP_NAME)/; \
		done; \
	done
	@echo "  binaries in $(DIST_DIR)/"

run: build ## Build and run locally
	@$(BIN_DIR)/$(APP_NAME)

generate: ## Run go generate
	$(GO) generate ./...

tidy: ## Tidy go modules
	$(GO) mod tidy
	$(GO) mod verify

# ── Test ──────────────────────────────────────────────────────────────────────
test: ## Run unit tests
	$(GO) test $(GOFLAGS) ./... -count=1 -timeout=120s

test-race: ## Run tests with race detector
	$(GO) test $(GOFLAGS) -race ./... -count=1 -timeout=120s

test-cover: ## Run tests with coverage report
	@mkdir -p $(COVER_DIR)
	$(GO) test $(GOFLAGS) ./... \
		-coverprofile=$(COVER_DIR)/coverage.out \
		-covermode=atomic \
		-count=1 -timeout=120s
	$(GO) tool cover -html=$(COVER_DIR)/coverage.out \
		-o $(COVER_DIR)/coverage.html
	$(GO) tool cover -func=$(COVER_DIR)/coverage.out \
		| grep total \
		| awk '{print "  coverage:", $$3}'
	@open $(COVER_DIR)/coverage.html 2>/dev/null || true

test-integration: ## Run integration tests (requires running services)
	$(GO) test $(GOFLAGS) -tags=integration ./... -count=1 -timeout=300s

bench: ## Run benchmarks
	$(GO) test -bench=. -benchmem ./... -run=^$

# ── Lint / Format ──────────────────────────────────────────────────────────
$(GOLANGCI_LINT): ## Download golangci-lint
	@mkdir -p $(BIN_DIR)
	@curl -sSfL \
		https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh \
		| sh -s -- -b $(BIN_DIR) $(GOLANGCI_LINT_VERSION)

lint: $(GOLANGCI_LINT) ## Run golangci-lint
	$(GOLANGCI_LINT) run --timeout=5m

lint-fix: $(GOLANGCI_LINT) ## Run golangci-lint with auto-fix
	$(GOLANGCI_LINT) run --fix

fmt: ## Format Go code
	$(GO) fmt ./...
	goimports -w $(GO_FILES) 2>/dev/null || true

vet: ## Run go vet
	$(GO) vet ./...

check: fmt vet lint ## Run all checks (fmt + vet + lint)

# ── Docker ────────────────────────────────────────────────────────────────────
docker-build: ## Build Docker image
	docker build \
		--build-arg VERSION=$(VERSION) \
		--build-arg GIT_COMMIT=$(GIT_COMMIT) \
		--build-arg BUILD_TIME=$(BUILD_TIME) \
		--tag $(IMAGE):$(VERSION) \
		--tag $(IMAGE):latest \
		.
	@echo "  built $(IMAGE):$(VERSION)"

docker-push: ## Push Docker image to registry
	docker push $(IMAGE):$(VERSION)
	docker push $(IMAGE):latest

docker-run: ## Run Docker container locally
	docker run --rm -it \
		--env-file .env \
		-p 8080:8080 \
		$(IMAGE):latest

# ── Deps / Tools ──────────────────────────────────────────────────────────────
deps: ## Download module dependencies
	$(GO) mod download

# ── Clean ─────────────────────────────────────────────────────────────────────
clean: ## Remove build artifacts and caches
	@rm -rf $(BIN_DIR) $(DIST_DIR) $(COVER_DIR)
	$(GO) clean -testcache
	@echo "  cleaned"

# ── Help ──────────────────────────────────────────────────────────────────────
help: ## Display this help
	@awk 'BEGIN {FS = ":.*##"; printf "\nUsage:\n  make \033[36m<target>\033[0m\n\nTargets:\n"} \
	/^[a-zA-Z_0-9-]+:.*?##/ { printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2 }' \
	$(MAKEFILE_LIST)
```

---

## 12. Docker & Container Workflows

```makefile
# ============================================================================
# Docker targets
# ============================================================================

REGISTRY    ?= docker.io/myorg
APP_NAME    ?= myapp
VERSION     ?= $(shell git describe --tags --always --dirty)
IMAGE       := $(REGISTRY)/$(APP_NAME)
PLATFORMS   := linux/amd64,linux/arm64

# Build for current platform
.PHONY: docker-build
docker-build: ## Build Docker image
	docker build \
		--file Dockerfile \
		--build-arg VERSION=$(VERSION) \
		--build-arg GIT_COMMIT=$(GIT_COMMIT) \
		--cache-from $(IMAGE):latest \
		--tag $(IMAGE):$(VERSION) \
		--tag $(IMAGE):latest \
		.

# Multi-platform build with buildx
.PHONY: docker-buildx
docker-buildx: ## Build multi-platform image with buildx
	docker buildx build \
		--platform $(PLATFORMS) \
		--file Dockerfile \
		--build-arg VERSION=$(VERSION) \
		--tag $(IMAGE):$(VERSION) \
		--tag $(IMAGE):latest \
		--push \
		.

# Push specific or both tags
.PHONY: docker-push docker-push-latest
docker-push: ## Push versioned image tag
	docker push $(IMAGE):$(VERSION)

docker-push-latest: ## Push versioned + latest tags
	docker push $(IMAGE):$(VERSION)
	docker push $(IMAGE):latest

# Tag an existing image
.PHONY: docker-tag
docker-tag: ## Re-tag image (use: make docker-tag FROM=old TO=new)
	docker tag $(IMAGE):$(FROM) $(IMAGE):$(TO)

# Run container with local .env
.PHONY: docker-run
docker-run: ## Run container locally with .env file
	docker run --rm -it \
		--name $(APP_NAME)-dev \
		--env-file .env \
		-p 8080:8080 \
		-v $(CURDIR)/data:/data \
		$(IMAGE):$(VERSION)

# Shell into running container
.PHONY: docker-shell
docker-shell: ## Open shell in running container
	docker exec -it $(APP_NAME)-dev /bin/sh

# Docker Compose targets
.PHONY: up down logs ps
up: ## Start all services (docker compose)
	docker compose up -d
	@echo "  services started — run 'make logs' to follow"

down: ## Stop all services
	docker compose down

logs: ## Tail service logs
	docker compose logs -f --tail=100

ps: ## Show service status
	docker compose ps

# Compose with specific service
.PHONY: up-db up-redis
up-db: ## Start only the database
	docker compose up -d postgres

up-redis: ## Start only Redis
	docker compose up -d redis

# Build and push in one step
.PHONY: release
release: docker-build docker-push-latest ## Build and push image

# Scan image for vulnerabilities
.PHONY: docker-scan
docker-scan: ## Scan image with trivy
	trivy image --exit-code 1 --severity HIGH,CRITICAL $(IMAGE):$(VERSION)

# Clean up Docker resources
.PHONY: docker-clean
docker-clean: ## Remove project images and stopped containers
	docker compose down --remove-orphans 2>/dev/null || true
	docker images --filter "reference=$(IMAGE)*" -q \
		| xargs -r docker rmi -f
	docker system prune -f --filter "label=project=$(APP_NAME)"
```

---

## 13. Terraform / Infrastructure Makefile

```makefile
# ============================================================================
# Terraform targets
# ============================================================================

TF_DIR       ?= terraform
ENVIRONMENT  ?= development
TF_VARS_FILE := environments/$(ENVIRONMENT).tfvars
TF_STATE_KEY := $(ENVIRONMENT)/terraform.tfstate
TF_PLAN_FILE := .terraform/$(ENVIRONMENT).tfplan

# Terraform binary (allow override for version pinning)
TF           := terraform

.PHONY: tf-init tf-validate tf-fmt tf-plan tf-apply tf-destroy \
        tf-output tf-state-list tf-unlock tf-clean

# Guard for required variables
guard-%:
	@test -n "$($*)" || (echo "ERROR: $* is required"; exit 1)

# Init — run when first using or when providers change
tf-init: ## Initialize Terraform (run first or after provider changes)
	@echo "Initializing Terraform for $(ENVIRONMENT)..."
	cd $(TF_DIR) && $(TF) init \
		-backend-config="key=$(TF_STATE_KEY)" \
		-reconfigure

tf-validate: ## Validate Terraform configuration
	cd $(TF_DIR) && $(TF) validate

tf-fmt: ## Format Terraform files
	$(TF) fmt -recursive $(TF_DIR)

tf-fmt-check: ## Check Terraform formatting (CI)
	$(TF) fmt -check -recursive $(TF_DIR)

# Plan — always review before apply
tf-plan: tf-validate ## Generate Terraform plan
	@echo "Planning for $(ENVIRONMENT)..."
	cd $(TF_DIR) && $(TF) plan \
		-var-file=$(TF_VARS_FILE) \
		-out=$(TF_PLAN_FILE) \
		-detailed-exitcode
	@echo "  plan saved to $(TF_PLAN_FILE)"

# Apply — use saved plan
tf-apply: guard-ENVIRONMENT ## Apply saved Terraform plan
	@echo "Applying plan for $(ENVIRONMENT)..."
	@read -p "Apply to $(ENVIRONMENT)? [y/N] " confirm && [ "$$confirm" = "y" ]
	cd $(TF_DIR) && $(TF) apply $(TF_PLAN_FILE)

# Apply with auto-approve (CI use only)
tf-apply-ci: ## Apply saved plan without confirmation (CI only)
	cd $(TF_DIR) && $(TF) apply -auto-approve $(TF_PLAN_FILE)

# Destroy (with confirmation)
tf-destroy: guard-ENVIRONMENT ## Destroy all resources (dangerous!)
	@echo "WARNING: This will destroy all resources in $(ENVIRONMENT)"
	@read -p "Type 'destroy $(ENVIRONMENT)' to confirm: " confirm \
		&& [ "$$confirm" = "destroy $(ENVIRONMENT)" ]
	cd $(TF_DIR) && $(TF) destroy \
		-var-file=$(TF_VARS_FILE) \
		-auto-approve

tf-output: ## Show Terraform outputs
	cd $(TF_DIR) && $(TF) output -json | jq .

tf-state-list: ## List resources in Terraform state
	cd $(TF_DIR) && $(TF) state list

tf-state-show: ## Show specific resource (use: make tf-state-show RESOURCE=aws_instance.web)
	cd $(TF_DIR) && $(TF) state show $(RESOURCE)

tf-unlock: guard-LOCK_ID ## Force-unlock state (use: make tf-unlock LOCK_ID=xxx)
	cd $(TF_DIR) && $(TF) force-unlock $(LOCK_ID)

tf-clean: ## Remove Terraform cache and plan files
	find $(TF_DIR) -name '.terraform' -type d | xargs rm -rf
	find $(TF_DIR) -name '*.tfplan' -delete
	find $(TF_DIR) -name '.terraform.lock.hcl' -delete

# Environment shortcuts
.PHONY: plan-dev plan-staging plan-prod apply-dev apply-staging apply-prod

plan-dev:
	ENVIRONMENT=development $(MAKE) tf-plan

plan-staging:
	ENVIRONMENT=staging $(MAKE) tf-plan

plan-prod:
	ENVIRONMENT=production $(MAKE) tf-plan

apply-dev: guard-TF_CONFIRMED
	ENVIRONMENT=development $(MAKE) tf-apply-ci

apply-staging: guard-TF_CONFIRMED
	ENVIRONMENT=staging $(MAKE) tf-apply-ci

apply-prod: guard-TF_CONFIRMED
	ENVIRONMENT=production $(MAKE) tf-apply-ci
```

---

## 14. Kubernetes & Helm Makefile

```makefile
# ============================================================================
# Kubernetes and Helm targets
# ============================================================================

CLUSTER      ?= dev-cluster
NAMESPACE    ?= default
APP_NAME     ?= myapp
CHART_DIR    := helm/$(APP_NAME)
HELM_RELEASE := $(APP_NAME)
IMAGE_TAG    ?= $(shell git describe --tags --always --dirty)
KUBECONFIG   ?= $(HOME)/.kube/config

KUBECTL      := kubectl --context=$(CLUSTER) --namespace=$(NAMESPACE)
HELM         := helm --kube-context=$(CLUSTER)

.PHONY: k8s-apply k8s-delete k8s-status k8s-logs k8s-exec \
        helm-lint helm-template helm-diff helm-deploy helm-rollback \
        kube-ctx ns-create

# ── kubectl ───────────────────────────────────────────────────────────────────
k8s-apply: ## Apply Kubernetes manifests
	$(KUBECTL) apply -f manifests/

k8s-apply-dry: ## Dry-run apply Kubernetes manifests
	$(KUBECTL) apply --dry-run=client -f manifests/

k8s-delete: ## Delete Kubernetes resources
	$(KUBECTL) delete -f manifests/

k8s-status: ## Show rollout status
	$(KUBECTL) rollout status deployment/$(APP_NAME) --timeout=120s

k8s-restart: ## Restart deployment (rolling restart)
	$(KUBECTL) rollout restart deployment/$(APP_NAME)

k8s-rollback: ## Rollback to previous deployment
	$(KUBECTL) rollout undo deployment/$(APP_NAME)

k8s-logs: ## Tail pod logs
	$(KUBECTL) logs -f \
		-l app.kubernetes.io/name=$(APP_NAME) \
		--all-containers=true \
		--max-log-requests=10 \
		--tail=100

k8s-exec: ## Exec into pod (use: make k8s-exec CMD=/bin/sh)
	$(KUBECTL) exec -it \
		$$($(KUBECTL) get pod -l app.kubernetes.io/name=$(APP_NAME) \
			-o jsonpath='{.items[0].metadata.name}') \
		-- $(or $(CMD),/bin/sh)

k8s-port-forward: ## Port-forward to app (use: make k8s-port-forward LOCAL=8080 REMOTE=8080)
	$(KUBECTL) port-forward \
		svc/$(APP_NAME) \
		$(or $(LOCAL),8080):$(or $(REMOTE),8080)

k8s-describe: ## Describe the deployment
	$(KUBECTL) describe deployment/$(APP_NAME)

k8s-events: ## Show recent events in namespace
	$(KUBECTL) get events --sort-by=.lastTimestamp | tail -30

ns-create: ## Create namespace if it doesn't exist
	$(KUBECTL) get namespace $(NAMESPACE) 2>/dev/null \
		|| $(KUBECTL) create namespace $(NAMESPACE)

kube-ctx: ## Switch kubectl context
	kubectl config use-context $(CLUSTER)

# ── Helm ──────────────────────────────────────────────────────────────────────
helm-deps: ## Update Helm chart dependencies
	$(HELM) dependency update $(CHART_DIR)

helm-lint: helm-deps ## Lint the Helm chart
	$(HELM) lint $(CHART_DIR) \
		-f $(CHART_DIR)/values.yaml \
		-f $(CHART_DIR)/values-$(NAMESPACE).yaml 2>/dev/null || true

helm-template: ## Render Helm templates locally
	$(HELM) template $(HELM_RELEASE) $(CHART_DIR) \
		--namespace $(NAMESPACE) \
		-f $(CHART_DIR)/values.yaml \
		--set image.tag=$(IMAGE_TAG)

helm-diff: ## Show diff between deployed and local chart (requires helm-diff plugin)
	$(HELM) diff upgrade $(HELM_RELEASE) $(CHART_DIR) \
		--namespace $(NAMESPACE) \
		-f $(CHART_DIR)/values.yaml \
		--set image.tag=$(IMAGE_TAG)

helm-deploy: helm-lint ## Deploy Helm chart
	@echo "Deploying $(HELM_RELEASE) to $(CLUSTER)/$(NAMESPACE)..."
	$(HELM) upgrade --install $(HELM_RELEASE) $(CHART_DIR) \
		--namespace $(NAMESPACE) \
		--create-namespace \
		-f $(CHART_DIR)/values.yaml \
		--set image.tag=$(IMAGE_TAG) \
		--atomic \
		--timeout=5m \
		--wait
	@echo "  deployed $(HELM_RELEASE):$(IMAGE_TAG)"

helm-rollback: ## Rollback Helm release (use: make helm-rollback REV=2)
	$(HELM) rollback $(HELM_RELEASE) $(or $(REV),0) --namespace $(NAMESPACE)

helm-status: ## Show Helm release status
	$(HELM) status $(HELM_RELEASE) --namespace $(NAMESPACE)

helm-history: ## Show Helm release history
	$(HELM) history $(HELM_RELEASE) --namespace $(NAMESPACE)

helm-uninstall: ## Uninstall Helm release
	$(HELM) uninstall $(HELM_RELEASE) --namespace $(NAMESPACE)

helm-values: ## Show effective values for deployed release
	$(HELM) get values $(HELM_RELEASE) --namespace $(NAMESPACE)

# ── Full deploy workflow ──────────────────────────────────────────────────────
.PHONY: deploy deploy-staging deploy-production

deploy: docker-build docker-push helm-deploy ## Build, push, and deploy

deploy-staging: ## Deploy to staging
	$(MAKE) deploy CLUSTER=staging-cluster NAMESPACE=staging IMAGE_TAG=$(IMAGE_TAG)

deploy-production: guard-IMAGE_TAG ## Deploy to production (requires IMAGE_TAG)
	@read -p "Deploy $(IMAGE_TAG) to production? [y/N] " c && [ "$$c" = "y" ]
	$(MAKE) deploy \
		CLUSTER=prod-cluster \
		NAMESPACE=production \
		IMAGE_TAG=$(IMAGE_TAG)
```

---

## 15. Common Gotchas & Best Practices

### TAB vs spaces — the classic trap

```makefile
# ALWAYS use TAB (ASCII 0x09) for recipe indentation
# NEVER use spaces — Make will give a confusing error:
# *** missing separator.  Stop.

target:
	@echo "this line uses a TAB"    # ✅ correct
    @echo "this uses spaces"        # ❌ make: missing separator

# In vim: :set list shows tabs as ^I
# In VS Code: install "Makefile Tools" extension
# In git: git diff shows tabs as ^I
```

### Variables in recipes — `$` escaping

```makefile
# Make expands $VAR in recipes BEFORE passing to shell
# Use $$ to get a literal $ in the shell command

print-shell-var:
	@echo $$HOME          # shell $HOME — correct
	@echo $HOME           # Make variable HOME — probably empty

loop-example:
	@for i in 1 2 3; do \
		echo $$i; \        # $$i → $i in shell
	done

# Make variable reference
MYVAR := hello
print-make-var:
	@echo $(MYVAR)        # hello — Make expands before shell sees it
```

### Recursive `$(MAKE)` — not `make`

```makefile
# ALWAYS use $(MAKE) not 'make' when calling make from a recipe
# $(MAKE) passes flags (-j, -n, --dry-run) to the subprocess

# WRONG
sub-build:
	cd subdir && make build

# CORRECT
sub-build:
	cd subdir && $(MAKE) build

# Or use -C flag
sub-build:
	$(MAKE) -C subdir build
```

### Order-only prerequisites

```makefile
# Normal prerequisite: if bin/ changes, rebuild target
# Order-only (|): create bin/ if missing, but don't rebuild target when it changes

bin/app: cmd/main.go | bin    # bin/ created if missing, no rebuild on bin/ change
	go build -o $@ ./cmd/

bin:
	@mkdir -p $@
```

### `.PHONY` for everything non-file

```makefile
# If a file named 'test' exists, 'make test' does nothing without .PHONY
# Always declare task targets as .PHONY

.PHONY: test
test:
	go test ./...

# touch test    ← without .PHONY, this would break 'make test'
```

### Parallel execution safety

```makefile
# make -j4 runs 4 targets simultaneously — can cause race conditions

# Mark targets that must run sequentially
.NOTPARALLEL: deploy rollback migrate

# Use dependencies to enforce order
deploy: build test docker-push    # test and docker-push may run in parallel
	kubectl apply -f manifests/

# Force sequential with explicit ordering
sequential: step1
step1: ; @echo "step 1"
step2: step1 ; @echo "step 2"
step3: step2 ; @echo "step 3"
sequential: step3
```

### Dry run support

```makefile
# Support --dry-run / -n flag by using $(MAKE) recursively
# Or implement your own DRY_RUN variable

DRY_RUN ?=
RUN     := $(if $(DRY_RUN),echo "[DRY RUN]",)

deploy:
	$(RUN) kubectl apply -f manifests/
	$(RUN) helm upgrade --install myapp ./helm/myapp

# Usage: make deploy DRY_RUN=1
```

---

## 16. Quick Reference Cheat Sheet

```makefile
# ─── RULE STRUCTURE ───────────────────────────────────────────────────────────
target: dep1 dep2       # target depends on dep1 and dep2
[TAB]   @command        # @ suppress echo
[TAB]   -command        # - ignore errors
[TAB]   recipe          # plain — echo + run

target: dep1 | dir/     # | = order-only prerequisite

# ─── VARIABLE ASSIGNMENT ──────────────────────────────────────────────────────
VAR  = value            # recursive — expanded at use time
VAR := value            # simple — expanded at assignment time (prefer this)
VAR ?= value            # conditional — set only if not already set
VAR += more             # append
VAR != cmd              # shell assignment (GNU Make 4.0+)
VAR := $(shell cmd)     # shell assignment (all versions)

# ─── VARIABLE REFERENCE ───────────────────────────────────────────────────────
$(VAR)   ${VAR}         # variable reference
$$       in recipe      # literal $ passed to shell (not Make expansion)

# ─── AUTOMATIC VARIABLES ──────────────────────────────────────────────────────
$@   target name
$<   first prerequisite
$^   all prerequisites (deduped)
$?   prerequisites newer than target
$*   stem (matched part in pattern rule)
$(@D) $(@F)  directory / file part of target
$(<D) $(<F)  directory / file part of first prereq

# ─── PATTERN RULES ────────────────────────────────────────────────────────────
%.o: %.c                # compile any .c to .o
bin/%: cmd/%/main.go    # build any binary from cmd/

# ─── FUNCTIONS ────────────────────────────────────────────────────────────────
$(subst from,to,text)
$(patsubst pat,rep,text)    $(SRCS:%.go=%.o)   # shorthand
$(filter pat,list)
$(filter-out pat,list)
$(sort list)
$(wildcard *.go)
$(dir path)   $(notdir path)   $(basename path)   $(suffix path)
$(addprefix pre,list)   $(addsuffix suf,list)
$(foreach var,list,text)
$(if cond,then,else)
$(or v1,v2,v3)
$(and v1,v2)
$(call name,arg1,arg2)
$(shell cmd)
$(error msg)   $(warning msg)   $(info msg)
$(origin var)  $(flavor var)

# ─── CONDITIONALS ────────────────────────────────────────────────────────────
ifeq ($(A),$(B))   ifneq ($(A),$(B))
ifdef VAR          ifndef VAR
else               endif

# ─── SPECIAL TARGETS ─────────────────────────────────────────────────────────
.PHONY: target          # not a file — always run
.DEFAULT_GOAL := help   # default target when none specified
.ONESHELL:              # all recipe lines in one shell
.SHELLFLAGS := -eu -o pipefail -c
SHELL := /bin/bash
.DELETE_ON_ERROR:       # delete target on recipe failure
.NOTPARALLEL: t1 t2     # don't parallelize these targets
```

### CLI flags

```bash
make                    # run default goal
make target             # run specific target
make t1 t2              # run multiple targets in order
make -n                 # dry-run: print commands, don't execute
make -B                 # force rebuild (ignore timestamps)
make -j4                # run 4 jobs in parallel
make -j                 # max parallelism (num CPUs)
make -C dir             # run make in directory
make -f file.mk         # use specific makefile
make VAR=value          # override variable
make --warn-undefined-variables   # warn on undefined $(VAR)
make --trace            # print rule provenance
make -p                 # print all rules and variables (debug)
make -d                 # full debug output
make --no-print-directory        # suppress "Entering directory" messages
```

### Key rules at a glance

| Rule | Detail |
|------|--------|
| TAB, not spaces | Recipe lines MUST use TAB — spaces cause "missing separator" error |
| `$$` in recipes | `$` for shell vars in recipes — `$HOME` is a Make var, `$$HOME` is shell |
| Always `.PHONY` | Non-file targets need `.PHONY` or they silently do nothing if a file exists |
| `:=` over `=` | Use simply-expanded `:=` for `$(shell ...)` — avoids re-running every reference |
| `$(MAKE)` not `make` | Recursive make calls must use `$(MAKE)` to propagate flags like `-j` and `-n` |
| `?=` for defaults | Lets callers override with `make TARGET VAR=value` |
| Guard pattern | `guard-%` target enforces required variables with clear error messages |
| Order-only `|` | `target: deps \| dirs` — create dirs if missing without triggering rebuild |
| `.DEFAULT_GOAL` | Set explicitly — don't rely on first target being the default |
| `@` suppress, `-` ignore | `@echo` = silent, `-rm` = don't fail if file missing |

---

*Part of DevOpsNotes / LANGUAGES — see also `07_Regex.md`, `09_QueryLanguages.md`*