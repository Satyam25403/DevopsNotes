# HashiCorp Vault - Introduction & Architecture

Understanding what Vault is, the secrets-sprawl problem it solves, and how it's architected under the hood.

## Table of Contents
- [What is Vault](#what-is-vault)
- [The Secrets Sprawl Problem](#the-secrets-sprawl-problem)
- [Why DevOps Teams Use It](#why-devops-teams-use-it)
- [Core Concepts Overview](#core-concepts-overview)
- [Architecture](#architecture)
- [The Seal/Unseal Concept](#the-sealunseal-concept)
- [Vault vs Alternatives](#vault-vs-alternatives)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## What is Vault

**HashiCorp Vault is a secrets management tool that centrally stores, controls access to, and dynamically generates sensitive data** — passwords, API keys, certificates, tokens, and encryption keys.

**Visual:**
```
Without Vault                          With Vault
┌─────────────────────┐                ┌──────────────────────┐
│ app config.yaml        │                │  Vault Server           │
│  db_password: "hunter2"│                │  ┌──────────────────┐  │
│                         │                │  │ Encrypted secrets   │  │
│ .env file                │                │  │ + access policies    │  │
│  API_KEY=sk_live_abc123  │   ───────→    │  └──────────────────┘  │
│                         │                └──────────┬───────────┘
│ Jenkins credentials       │                            │
│  (yet another copy)        │                 apps/CI request
│                              │                 secrets at runtime,
│ Secrets scattered,           │                 nothing hardcoded
│ duplicated, hard to rotate    │                 anywhere
└─────────────────────┘
```

---

## The Secrets Sprawl Problem

**Visual:**
```
Typical organization BEFORE centralized secrets management:

Secret: production database password
  ├── Copy in app's config file (committed to git by mistake once)
  ├── Copy in Jenkins credential store
  ├── Copy in a teammate's local .env file
  ├── Copy in a Kubernetes Secret (base64, NOT encrypted at rest)
  └── Copy in a Slack message from 8 months ago 😬

Problem: To rotate this password, you must find and update
EVERY one of these copies. Miss one → outage or security gap.
```

**Vault's answer:** one source of truth, accessed dynamically, never hardcoded, easy to rotate centrally.

---

## Why DevOps Teams Use It

**Visual:**
```
Problem                              How Vault Helps
──────────────────────────────────────────────────────────────────
Hardcoded secrets in code/configs    Apps fetch secrets at runtime via API
Secrets never rotated (too risky)    Dynamic secrets with short TTLs, auto-rotate
No audit trail of who accessed what  Every secret access is logged
Same DB password for all environments Unique dynamic credentials per app/lease
Manual, error-prone secret distribution Policy-based automated access control
Encryption logic duplicated per app   Transit engine centralizes "encryption as a service"
```

**Visual — Dynamic Secrets, Vault's signature feature:**
```
Traditional (Static) Secret:
Database password: "hunter2"  ← same password, forever, shared by everyone

Vault Dynamic Secret:
App requests DB credentials → Vault creates a UNIQUE, TEMPORARY
                                database user on the fly
                                     ↓
                          username: v-approle-abc123
                          password: A1b2C3-random-generated
                          TTL: 1 hour
                                     ↓
                          After 1 hour: Vault automatically
                          revokes this user from the database

Benefit: even if leaked, the credential is short-lived and
uniquely traceable to the specific app/request that requested it.
```

---

## Core Concepts Overview

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                      Vault Concepts                       │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Secrets Engine    →  Where/how secrets are stored or       │
│                       generated (KV, Database, PKI, etc.)   │
│                                                           │
│  Auth Method       →  HOW a client proves its identity       │
│                       (Token, AppRole, Kubernetes, LDAP)      │
│                                                           │
│  Policy            →  WHAT an authenticated identity          │
│                       is allowed to do (read/write paths)      │
│                                                           │
│  Token             →  The credential issued after successful   │
│                       auth, used for subsequent requests          │
│                                                           │
│  Lease             →  Time-bound validity of a secret            │
│                       (auto-expires, can be renewed/revoked)      │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

## Architecture

**Visual:**
```
                     ┌───────────────────────────────┐
                     │           Vault Server            │
                     │  ┌───────────────────────────┐  │
                     │  │   HTTP API / CLI Interface   │  │
                     │  ├───────────────────────────┤  │
                     │  │   Auth Methods (identity)      │  │
                     │  ├───────────────────────────┤  │
                     │  │   Policies (authorization)      │  │
                     │  ├───────────────────────────┤  │
                     │  │   Secrets Engines                │  │
                     │  │   (KV, Database, PKI, Transit)     │  │
                     │  ├───────────────────────────┤  │
                     │  │   Barrier (encryption layer)        │  │
                     │  └─────────────┬─────────────┘  │
                     └────────────────┼─────────────────┘
                                      ↓
                     ┌───────────────────────────────┐
                     │      Storage Backend               │
                     │   (Raft / Consul / etc — stores       │
                     │    only ENCRYPTED data, never plaintext)│
                     └───────────────────────────────┘
```

**Flow in words:**
1. A client (app, human, CI pipeline) authenticates via an **Auth Method**.
2. Vault checks the attached **Policy** to determine what that identity can access.
3. The request hits a **Secrets Engine**, which either retrieves a stored secret or dynamically generates one.
4. All data at rest passes through the **Barrier** — an encryption layer — before touching the **Storage Backend**. Storage never sees plaintext.

---

## The Seal/Unseal Concept

**This is one of Vault's most distinctive design choices.**

**Visual:**
```
Sealed Vault (default state after any restart):
┌────────────────────────────┐
│  Vault Server                  │
│  ┌──────────────────────┐    │
│  │  Encrypted Storage        │    │  ← Vault CANNOT read this yet
│  └──────────────────────┘    │      (it doesn't have the master key
│  Status: SEALED 🔒              │       in memory)
└────────────────────────────┘

Unsealing Process (Shamir's Secret Sharing):
Master Key is SPLIT into N key shares (e.g. 5 shares)
Any THRESHOLD number (e.g. 3 of 5) must be provided to reconstruct it

Key Share 1 ─┐
Key Share 2 ─┼──→ Combine 3 of 5 ──→ Master Key reconstructed
Key Share 3 ─┘         (in memory only, never stored)
                            ↓
                  Vault Server: UNSEALED 🔓
                  (can now decrypt storage, serve requests)
```

**Why split the key across multiple people?** No single person (or compromised machine) can unseal Vault alone — it requires cooperation from a threshold number of trusted key-holders, preventing any single point of compromise.

**Auto-unseal (production pattern):** Instead of humans manually entering key shares after every restart, cloud KMS services (AWS KMS, Azure Key Vault, GCP KMS) can hold the master key and unseal automatically — covered in file 05.

---

## Vault vs Alternatives

**Visual:**
```
Tool                    Focus                        Notes
─────────────────────────────────────────────────────────────────────
HashiCorp Vault          General secrets management     Dynamic secrets, multi-cloud,
                                                          most feature-rich, self-hosted
AWS Secrets Manager       AWS-native secrets                Simple, tightly AWS-integrated
Azure Key Vault           Azure-native secrets                Simple, tightly Azure-integrated
Kubernetes Secrets         K8s-native, base64 only              NOT encrypted at rest by default,
                                                          not a real secrets manager alone
SOPS                      Encrypt secrets IN git files          No dynamic secrets, no central server

Vault's niche: cloud-agnostic, supports dynamic/rotating
secrets across databases, cloud providers, and PKI — the
most powerful option when secrets span multiple platforms.
```

---

## Real-Life DevOps Use Case

**Scenario:** A company runs services across AWS and on-prem, with each team historically managing its own database passwords and API keys in scattered `.env` files and Kubernetes Secrets.

**Without Vault:**
- A departing employee's laptop had access to production database credentials that were never rotated afterward.
- No one can answer "who accessed the payment API key last month?" during a security audit.

**What the DevOps team does:**
1. Deploys a **highly available Vault cluster** using the Raft storage backend across 3 nodes.
2. Configures **Kubernetes auth method** so pods authenticate to Vault using their service account tokens — no hardcoded credentials in any container image or config.
3. Migrates database credentials to the **Database secrets engine**, so each pod gets a unique, short-lived (1-hour TTL) database user instead of a shared static password.
4. Defines **Policies** so the `payments` team's pods can only read `secret/payments/*` and generate database credentials for the payments DB — not any other team's secrets.
5. Enables **audit logging** so every secret access is recorded, satisfying the security audit requirement.
6. When an employee leaves, no manual "go find every place their access lives" — access was always scoped via short-lived, automatically-issued Vault leases, not personally-known static passwords.

**Result:** Secrets become centrally managed, automatically rotated, tightly scoped per team, and fully auditable — replacing scattered, static, manually-managed credentials.

---

Next: **01installation_and_setup.md** — installing Vault, running dev and production servers, initializing and unsealing, and storing your first secret.