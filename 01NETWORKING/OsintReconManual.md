# OSINT & Reconnaissance — Red Team Field Manual
### Passive Recon | Maltego | Shodan | recon-ng | theHarvester | Subdomain Enum | Target Profiling

> **Series Position:** Module 14
> Cross-references: `Web_Application_Security_RedTeam_Field_Manual.md` (initial recon before web testing), `Active_Directory_RedTeam_Field_Manual.md` (user enumeration feeds into AD attacks), `Wireless_Security_RedTeam_Field_Manual.md` (OSINT identifies wireless infrastructure), `Social_Engineering` (OSINT feeds phishing campaigns).
>
> **Red Team Lens:** OSINT is the phase that separates professional red teamers from script kiddies. Every hour spent in reconnaissance saves three hours of failed exploitation. You learn the org's technology stack, employee naming conventions, email formats, exposed infrastructure, leaked credentials, and attack surface — all without touching a single system. Done right, OSINT is completely invisible and provides the intelligence to make every subsequent attack precision-targeted.
>
> **Ethics & Legal:** All techniques in this module use only publicly available information. OSINT against publicly accessible information is legal. However: (1) Always operate within your engagement scope — OSINT on employees, subsidiaries, or related companies outside your written authorization can create legal liability. (2) Some automated tools make network requests that could be logged. (3) Never create fake profiles or impersonate individuals to gather information.

---

## Table of Contents

### PART 1 — OSINT MINDSET & METHODOLOGY
1. [The OSINT Methodology — Intelligence Cycle](#1-the-osint-methodology--intelligence-cycle)
2. [OSINT Framework — Tool Map](#2-osint-framework--tool-map)
3. [Setting Up an Anonymized OSINT Environment](#3-setting-up-an-anonymized-osint-environment)

### PART 2 — DOMAIN & DNS INTELLIGENCE
4. [Domain Registration & WHOIS Analysis](#4-domain-registration--whois-analysis)
5. [DNS Enumeration — The Full Picture](#5-dns-enumeration--the-full-picture)
6. [Subdomain Enumeration — Finding Hidden Infrastructure](#6-subdomain-enumeration--finding-hidden-infrastructure)
7. [Certificate Transparency — SSL as Intel Source](#7-certificate-transparency--ssl-as-intel-source)
8. [DNS History & Zone Transfer Attempts](#8-dns-history--zone-transfer-attempts)

### PART 3 — INFRASTRUCTURE INTELLIGENCE
9. [Shodan — The Internet's Search Engine](#9-shodan--the-internets-search-engine)
10. [Censys — Certificate & Infrastructure Search](#10-censys--certificate--infrastructure-search)
11. [ASN & IP Range Discovery](#11-asn--ip-range-discovery)
12. [Cloud Infrastructure Fingerprinting](#12-cloud-infrastructure-fingerprinting)
13. [Web Technology Fingerprinting](#13-web-technology-fingerprinting)

### PART 4 — PEOPLE & ORGANIZATIONAL INTELLIGENCE
14. [Employee Enumeration — LinkedIn, GitHub, OSINT](#14-employee-enumeration--linkedin-github-osint)
15. [Email Address Discovery & Harvesting](#15-email-address-discovery--harvesting)
16. [Email Format Identification](#16-email-format-identification)
17. [Phone Number & Physical Address Intelligence](#17-phone-number--physical-address-intelligence)

### PART 5 — CREDENTIAL & DATA LEAK INTELLIGENCE
18. [Breach Database Searching](#18-breach-database-searching)
19. [Paste Site Monitoring & History](#19-paste-site-monitoring--history)
20. [GitHub Secret Scanning — Public Repos](#20-github-secret-scanning--public-repos)
21. [Google Dorks — Advanced Search Operators](#21-google-dorks--advanced-search-operators)

### PART 6 — SPECIALIZED OSINT TOOLS
22. [theHarvester — Automated Email & Infrastructure](#22-theharvester--automated-email--infrastructure)
23. [recon-ng — Modular OSINT Framework](#23-recon-ng--modular-osint-framework)
24. [Maltego — Visual Link Analysis](#24-maltego--visual-link-analysis)
25. [SpiderFoot — Automated OSINT](#25-spiderfoot--automated-osint)
26. [Amass — Comprehensive Attack Surface Mapping](#26-amass--comprehensive-attack-surface-mapping)

### PART 7 — SOCIAL MEDIA & WEB INTELLIGENCE
27. [LinkedIn Intelligence](#27-linkedin-intelligence)
28. [Twitter/X Intelligence](#28-twitterx-intelligence)
29. [Wayback Machine & Web Archives](#29-wayback-machine--web-archives)
30. [Job Listings as Intel Source](#30-job-listings-as-intel-source)

### PART 8 — BUILDING THE TARGET PROFILE
31. [Aggregating Intel — Building the Complete Picture](#31-aggregating-intel--building-the-complete-picture)
32. [Attack Surface Map from OSINT](#32-attack-surface-map-from-osint)
33. [OSINT-Driven Attack Planning](#33-osint-driven-attack-planning)

---

# PART 1 — OSINT MINDSET & METHODOLOGY

---

## 1. The OSINT Methodology — Intelligence Cycle

### Layman's Terms
OSINT (Open Source Intelligence) means **finding information about a target using only publicly available sources** — no hacking, no unauthorized access, no scanning their systems. The internet contains an enormous amount of information about every organization: employee names and roles (LinkedIn), technical infrastructure (Shodan, DNS records), leaked credentials (breach databases), source code with hardcoded secrets (GitHub), and detailed technical specs in job postings. A skilled OSINT analyst can map an organization's entire attack surface before sending a single packet.

### Real-World Event
**The SolarWinds breach (2020)** — the threat actor (APT29) conducted extensive OSINT before the operation: mapping SolarWinds' development infrastructure, identifying the Orion update mechanism, understanding signing processes. This reconnaissance guided the supply chain attack. Separately, researchers have demonstrated that **a company's job postings alone** can reveal their entire security stack — "Must know Splunk, CrowdStrike, Palo Alto" tells an attacker exactly what they'll face. OSINT intelligence directly informs attack strategy.

```
THE INTELLIGENCE CYCLE:

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  1. PLANNING          → Define what you need to find        │
│         │               What is the engagement goal?        │
│         ▼               What attack vectors are in scope?   │
│  2. COLLECTION        → Gather raw data from sources        │
│         │               DNS, WHOIS, Shodan, LinkedIn, etc.  │
│         ▼                                                    │
│  3. PROCESSING        → Clean and organize the data         │
│         │               Remove duplicates, verify accuracy  │
│         ▼                                                    │
│  4. ANALYSIS          → Extract meaning from the data       │
│         │               What vulnerabilities does this      │
│         │               infrastructure suggest?             │
│         ▼               What attack paths exist?            │
│  5. DISSEMINATION     → Use the intelligence                │
│         │               Feed into phishing, exploitation,   │
│         │               social engineering planning         │
│         ▼                                                    │
│  6. FEEDBACK          → New questions arise, loop back      │
│                          Each finding opens new questions   │
└──────────────────────────────────────────────────────────────┘

OSINT COLLECTION LAYERS:

LAYER 1 — PASSIVE (completely invisible):
  Read publicly accessible web pages
  Query public databases (WHOIS, DNS)
  Read public social media profiles
  Search breach databases
  Download public documents
  → No contact with target systems
  → Zero footprint
  → Start here, stay here as long as possible

LAYER 2 — SEMI-PASSIVE (minimal footprint):
  DNS queries to target's nameservers
  HTTP requests to target's web servers
  Certificate transparency log queries
  → Appears as normal web traffic
  → Some requests logged by target

LAYER 3 — ACTIVE (visible):
  Port scanning target IP ranges
  Vulnerability scanning
  Directory brute-forcing
  → Clear fingerprint in target's logs
  → Only do with explicit authorization

RULE: For OSINT, stay in Layers 1-2.
      Layer 3 = reconnaissance scanning, not OSINT.
```

---

## 2. OSINT Framework — Tool Map

```
OSINT FRAMEWORK (osintframework.com — bookmark this):

TARGET CATEGORIES:
  ├── Username         → Namechk, Sherlock, whatsmyname
  ├── Email Address    → theHarvester, Hunter.io, Emailrep
  ├── Domain Name      → WHOIS, DNSdumpster, Amass, Subfinder
  ├── IP Address       → Shodan, Censys, VirusTotal, Robtex
  ├── Person           → Spokeo, Pipl, LinkedIn, social media
  ├── Phone Number     → Truecaller, numverify, carrier lookup
  ├── Image            → Google Reverse Image, TinEye, FaceCheck
  ├── Social Network   → Twint, Instaloader, LinkedIn
  ├── Company          → LinkedIn, Crunchbase, OpenCorporates
  ├── Document         → Google Dorks, Shodan, Scribd
  └── Geolocation      → Google Maps, Wigle, MaxMind

KEY TOOLS BY PHASE:

DOMAIN/INFRASTRUCTURE:
  amass          ← Most comprehensive subdomain enumeration
  subfinder      ← Fast passive subdomain discovery
  dnsx           ← DNS toolkit (resolve, brute, wildcard detect)
  massdns        ← Mass DNS resolution at scale
  shodan         ← Internet-connected device search
  censys         ← Certificate and infrastructure search
  whois          ← Domain registration info
  SecurityTrails ← Historical DNS, IP history
  
PEOPLE/EMAIL:
  theHarvester   ← Email, name, subdomain aggregation
  Hunter.io      ← Find emails at a domain (web/API)
  recon-ng       ← Modular OSINT framework
  linkedin2username ← Generate username list from LinkedIn
  Sherlock       ← Find username across all social platforms
  
CREDENTIALS:
  Have I Been Pwned (API) ← Check email in breach databases
  Dehashed        ← Search breached credential database
  LeakCheck       ← Credential search
  
CODE/SECRETS:
  truffleHog     ← Find secrets in git repos
  gitleaks       ← Secret scanning for git
  gitrob         ← Recon for sensitive files on GitHub
  
AUTOMATED PLATFORMS:
  SpiderFoot     ← Automated OSINT aggregation
  Maltego        ← Visual link analysis
  recon-ng       ← Python-based modular framework
  OSINT Industries ← Commercial OSINT platform
```

---

## 3. Setting Up an Anonymized OSINT Environment

```bash
# OPSEC FOR OSINT:
# Your queries are logged by many services.
# Some tools send identifying information in User-Agent, referer.
# For sensitive engagements, use an anonymized environment.

# ── OPTION 1: DEDICATED VM WITH VPN ────────────────────────────────
# Create isolated VM (Kali or Tails) for OSINT work
# Connect to VPN before any research
# Use a VPN provider that doesn't log (ProtonVPN, Mullvad)
# Never mix OSINT research with personal browsing

# ── OPTION 2: TAILS OS (strongest anonymity) ────────────────────────
# Tails: amnesic OS that routes ALL traffic through Tor
# Boot from USB → no traces left on host machine
# Download: https://tails.boum.org/

# ── OPTION 3: PROXYCHAINS + TOR ────────────────────────────────────
# For command-line OSINT tools:
sudo apt install tor
sudo service tor start

# /etc/proxychains4.conf:
# socks5 127.0.0.1 9050  (Tor SOCKS port)

# Run tools via Tor:
proxychains4 theHarvester -d target.com -b google
proxychains4 amass enum -d target.com

# ── BROWSER FINGERPRINT REDUCTION ──────────────────────────────────
# Firefox in private mode with uBlock Origin
# Disable: WebRTC (leaks real IP even with VPN)
# about:config → media.peerconnection.enabled → false
# Use random User-Agent extension
# Check: ipleak.net, browserleaks.com

# ── SOCK PUPPET ACCOUNTS ───────────────────────────────────────────
# For social media research requiring accounts:
# Create dedicated research accounts with no personal info
# Different email, different device, different IP
# Never connect personal and research identities
# Note: creating fake identities may violate platform ToS — check with client

# ── KEEPING NOTES ──────────────────────────────────────────────────
# Organize intelligence as you gather it:
mkdir -p ~/osint/{company,domains,emails,people,infrastructure,credentials}

# Document sources for every finding:
# [Source] [Date] [Finding] [URL/Evidence]
# Helps with report writing and fact verification

# Tool: CherryTree (structured note-taking for OSINT/pentest)
sudo apt install cherrytree
```

---

# PART 2 — DOMAIN & DNS INTELLIGENCE

---

## 4. Domain Registration & WHOIS Analysis

### Layman's Terms
When a company registers a domain, they submit contact information to a public database. WHOIS reveals **who owns a domain, when it was registered, who hosts the DNS, and often technical contact names and email addresses**. Even with privacy protection, metadata like registrar, nameservers, and registration dates is valuable.

```bash
# ── BASIC WHOIS ─────────────────────────────────────────────────────
whois target.com
# Expected output:
# Domain Name: TARGET.COM
# Registrar: GoDaddy.com, LLC
# Registrar WHOIS Server: whois.godaddy.com
# Updated Date: 2023-05-15
# Creation Date: 2005-03-20     ← Company founded around 2005?
# Registry Expiry Date: 2025-03-20  ← Expiry = potential phishing window
#
# Registrant Organization: Target Corporation
# Registrant State/Province: Minnesota
# Registrant Country: US
#
# Name Server: NS1.TARGETCORP.COM   ← Internal DNS = they run their own nameservers!
# Name Server: NS2.TARGETCORP.COM
#
# DNSSEC: unsigned   ← DNSSEC not implemented

# ── INTELLIGENCE EXTRACTED FROM WHOIS ──────────────────────────────
# Registrant email (if not privacy-protected):
#   admin@target.com → confirms email format: role@domain
#   john.smith@target.com → confirms format: firstname.lastname@domain
#
# Nameservers: ns1.target.com = self-hosted DNS (attack surface!)
#              ns1.amazonaws.com = AWS Route 53 (check for subdomain takeover)
#              ns1.cloudflare.com = behind Cloudflare
#
# Registration date: age of company, domain squatting opportunities
# Expiry date: near expiry = target for domain hijacking (malicious actors)
#
# Registrar: useful for social engineering (impersonate registrar to IT)

# ── MULTIPLE DOMAINS — FIND RELATED REGISTRATIONS ────────────────────
# If target.com registered with email admin@target.com:
# That same email may have registered other company domains!

# Search by email using WHOIS history tools:
# https://viewdns.info/reversewhois/
# https://domaintools.com (paid)
# https://www.whoisxmlapi.com

# Example: find all domains registered by the same entity
curl "https://viewdns.info/reversewhois/?q=admin%40target.com" 2>/dev/null | \
  grep -oP '(?<=<td>)[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}(?=</td>)' | sort -u

# ── CERTIFICATE REGISTRATIONS ──────────────────────────────────────
# Company may register many domains for subsidiaries, projects, M&A targets
# Find them all: covered in Section 7 (Certificate Transparency)
```

---

## 5. DNS Enumeration — The Full Picture

```bash
# ── BASIC DNS LOOKUPS ───────────────────────────────────────────────
# Get all standard DNS records:
dig target.com ANY +short
# Expected:
# 93.184.216.34    (A record — IPv4 address)
# 2606:2800:21f:...(AAAA — IPv6)
# target.com. 3600 MX 10 mail.target.com.  (mail server)
# target.com. 3600 NS ns1.target.com.      (nameservers)
# target.com. 3600 TXT "v=spf1 include:..."(SPF record)

# Get each record type explicitly:
dig target.com A     # IPv4 addresses
dig target.com AAAA  # IPv6 addresses
dig target.com MX    # Mail servers → reveals email infrastructure
dig target.com NS    # Nameservers
dig target.com TXT   # Text records → SPF, DMARC, verification tokens
dig target.com SOA   # Start of Authority → primary nameserver, admin email
dig target.com CNAME # Canonical names → often reveals cloud services

# ── EXTRACT INTELLIGENCE FROM DNS ──────────────────────────────────
# MX records reveal email provider:
dig target.com MX +short
# Expected possibilities:
# 10 mail.target.com       ← self-hosted mail server
# 10 aspmx.l.google.com    ← Google Workspace (G Suite)
# 10 target-com.mail.protection.outlook.com ← Microsoft 365
# 10 mxa.mailgun.org       ← Mailgun (transactional email)

# TXT records are a goldmine:
dig target.com TXT +short
# Expected:
# "v=spf1 include:_spf.google.com include:sendgrid.net ip4:203.0.113.0/24 ~all"
#   → Uses Google + SendGrid for email, also has own mail server range
#
# "MS=ms12345678"          ← Microsoft 365 domain verification
# "google-site-verification=..." ← Google Search Console
# "atlassian-domain-verification=..." ← Uses Atlassian (Jira/Confluence!)
# "apple-domain-verification=..."  ← Has Apple developer account
# "docusign=..."                   ← Uses DocuSign for contracts
# "zoho-verification=..."          ← Uses Zoho (maybe CRM/email)

# DMARC record (email security policy):
dig _dmarc.target.com TXT +short
# Expected:
# "v=DMARC1; p=none; rua=mailto:dmarc@target.com"
# p=none = no enforcement (phishing emails WON'T be rejected!)
# p=quarantine = suspicious emails go to spam
# p=reject = hard rejection of unauthenticated email
# p=none = easiest to spoof domain in phishing campaigns

# SPF analysis:
dig target.com TXT | grep spf
# "v=spf1 ip4:203.0.113.0/24 include:mailchimp.net -all"
# This reveals: their mail server IP range (203.0.113.0/24)!
# And marketing email provider (Mailchimp)

# DKIM selectors (find email signing keys):
# Common selectors: default, google, k1, mail, dkim, selector1, selector2
dig default._domainkey.target.com TXT +short
dig google._domainkey.target.com TXT +short
dig selector1._domainkey.target.com TXT +short
# Expected:
# "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNA..."
# The selector tells you which email system uses it

# ── SOA RECORD (ADMIN EMAIL) ───────────────────────────────────────
dig target.com SOA +short
# Expected:
# ns1.target.com. admin.target.com. 2024011601 3600 900 604800 300
#                 ↑
#                 This is admin@target.com (dots replaced by @)!
```

---

## 6. Subdomain Enumeration — Finding Hidden Infrastructure

### Layman's Terms
Every subdomain potentially represents a **different application, service, or server** — often less-secured than the main site. dev.target.com might be the development server with debug mode on. staging.target.com might have real credentials. vpn.target.com reveals they have a VPN. mail.target.com reveals their mail server. admin.target.com might be directly accessible. Subdomain enumeration maps the full technical attack surface.

```bash
# ══════════════════════════════════════════════════════════════════
# PASSIVE SUBDOMAIN ENUMERATION (no DNS queries to target)
# ══════════════════════════════════════════════════════════════════

# ── SUBFINDER (best passive tool) ──────────────────────────────────
# Install:
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
# Or: sudo apt install subfinder

subfinder -d target.com -all -o subdomains_passive.txt
# -all = use all sources (certificate transparency, DNS datasets, etc.)
# Expected:
# mail.target.com
# dev.target.com
# staging.target.com
# vpn.target.com
# api.target.com
# jira.target.com
# confluence.target.com
# gitlab.target.com
# jenkins.target.com    ← CI/CD system!
# sso.target.com        ← Single Sign-On portal!

# ── AMASS PASSIVE (comprehensive, slower) ──────────────────────────
amass enum -passive -d target.com -o amass_passive.txt
# Queries: CertSpotter, Censys, CertDB, HackerTarget, Riddler, SecurityTrails, etc.

# ── CERTIFICATE TRANSPARENCY (Section 7 covers in depth) ──────────
curl -s "https://crt.sh/?q=%25.target.com&output=json" 2>/dev/null | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
names = set()
for cert in data:
    for name in cert.get('name_value','').split('\n'):
        name = name.strip().lstrip('*.')
        if name:
            names.add(name)
print('\n'.join(sorted(names)))
" > ct_subdomains.txt

cat ct_subdomains.txt
# Expected — all subdomains ever certified:
# api.target.com
# dev.target.com
# internal.target.com   ← "internal" — probably interesting!
# uat.target.com        ← User Acceptance Testing
# old.target.com        ← Old version — potentially unpatched!

# ══════════════════════════════════════════════════════════════════
# ACTIVE SUBDOMAIN ENUMERATION (queries go to DNS/target)
# ══════════════════════════════════════════════════════════════════

# ── DNS BRUTE FORCE (dnsx + wordlist) ──────────────────────────────
# Good wordlists:
# SecLists: /usr/share/seclists/Discovery/DNS/
ls /usr/share/seclists/Discovery/DNS/
# subdomains-top1million-5000.txt  ← Start with this
# subdomains-top1million-110000.txt ← More comprehensive
# dns-Jhaddix.txt                   ← Popular all-in-one

# Install dnsx:
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest

# Brute force with wordlist:
dnsx -d target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -r 8.8.8.8,1.1.1.1 \
  -o subdomains_bruteforce.txt \
  -t 50     # 50 threads
# Expected after running (2-5 minutes):
# dev.target.com → 192.168.100.50
# staging.target.com → 192.168.100.51
# api.target.com → 203.0.113.100

# ── COMBINED PIPELINE ──────────────────────────────────────────────
# Run passive + active, merge, resolve:
cat subdomains_passive.txt ct_subdomains.txt amass_passive.txt | sort -u > all_subdomains.txt
# Resolve which ones are live:
cat all_subdomains.txt | dnsx -r 8.8.8.8 -o live_subdomains.txt
# Check live subdomains for HTTP:
cat live_subdomains.txt | httpx -title -status-code -tech-detect -o web_services.txt
# Expected:
# dev.target.com [200] [Development Portal] [Apache/2.4]
# staging.target.com [302] [→ https://staging.target.com] []
# jenkins.target.com [200] [Jenkins] [Jenkins]  ← CI/CD exposed!
# jira.target.com [200] [Jira] [Atlassian Jira 8.20]

# ── INTERESTING SUBDOMAIN PATTERNS ────────────────────────────────
# High-value subdomains to look for:
grep -iE "dev|staging|uat|test|qa|sandbox" live_subdomains.txt
# → Development/test systems (often less secure, may have real data)

grep -iE "vpn|remote|access|gateway" live_subdomains.txt
# → VPN endpoints (credential brute-force target)

grep -iE "admin|manage|dashboard|portal" live_subdomains.txt
# → Admin panels (authentication target)

grep -iE "api|rest|graphql|ws" live_subdomains.txt
# → API endpoints (see API security module)

grep -iE "jenkins|gitlab|github|bitbucket|ci|cd|build" live_subdomains.txt
# → CI/CD systems (often have credentials/code)

grep -iE "jira|confluence|wiki|docs" live_subdomains.txt
# → Project management (employee info, internal docs)

grep -iE "mail|smtp|imap|webmail|owa" live_subdomains.txt
# → Mail infrastructure

grep -iE "sso|auth|login|idp|saml|oauth" live_subdomains.txt
# → Authentication systems (highest value targets)
```

---

## 7. Certificate Transparency — SSL as Intel Source

```bash
# Certificate Transparency (CT) logs record every TLS certificate issued.
# Companies can't issue a certificate secretly.
# → Every subdomain, internal system, cloud service with TLS = logged!

# ── CRT.SH — FREE CT LOG SEARCH ────────────────────────────────────
# Web UI: https://crt.sh/?q=%25.target.com

# API:
curl -s "https://crt.sh/?q=%25.target.com&output=json" | \
  python3 -c "
import json, sys
certs = json.load(sys.stdin)
print(f'Total certificates: {len(certs)}')
names = set()
for c in certs:
    for n in c.get('name_value','').split('\n'):
        n = n.strip().lstrip('*.')
        if '.' in n and not n.startswith('.'):
            names.add(n)
print(f'Unique subdomains: {len(names)}')
print('\n'.join(sorted(names)))
" 2>/dev/null

# ── EXTRACT INTEL FROM CERTIFICATES ───────────────────────────────
# Certificate contains: Subject Alternative Names (SANs)
# A wildcard cert *.target.com tells you they use subdomain heavily
# Multiple SANs reveal all services on one cert:
# *.target.com, *.api.target.com, *.internal.target.com
#                                   ↑
#                                   internal.target.com exists!

# Use curl to grab cert directly:
echo | openssl s_client -connect target.com:443 2>/dev/null | \
  openssl x509 -noout -text | grep -A20 "Subject Alternative Name"
# Expected:
# DNS:target.com
# DNS:www.target.com
# DNS:api.target.com
# DNS:auth.target.com
# DNS:internal.target.com  ← Internal system on public cert!

# ── FIND CERTIFICATES BY ORGANIZATION ──────────────────────────────
# Search by organization name (finds ALL their domains):
curl -s "https://crt.sh/?O=Target+Corporation&output=json" | \
  python3 -c "
import json, sys
certs = json.load(sys.stdin)
domains = set()
for c in certs:
    domains.add(c.get('common_name','').lstrip('*.'))
print('\n'.join(sorted(d for d in domains if d)))
"
# Expected: all domains the company has ever gotten a certificate for
# Reveals: subsidiaries, partner domains, internal domains, old domains

# ── GOOGLE CERTIFICATE TRANSPARENCY ────────────────────────────────
# Google's CT search (better uptime than crt.sh sometimes):
curl "https://transparencyreport.google.com/transparencyreport/api/v3/httpsreport/ct/certsearch?include_subdomains=true&domain=target.com"

# ── TIMING INTELLIGENCE FROM CERTS ────────────────────────────────
# Certificate issuance dates → when services were stood up
# Look for: certs issued right before public product launches
# Look for: certs for suspicious domains (M&A targets, new products)
curl -s "https://crt.sh/?q=%25.target.com&output=json" | \
  python3 -c "
import json, sys
certs = json.load(sys.stdin)
for c in sorted(certs, key=lambda x: x.get('entry_timestamp','')):
    print(f'{c[\"entry_timestamp\"][:10]}  {c[\"common_name\"]}')
" | tail -20
# Shows most recently issued certificates → new infrastructure!
```

---

## 8. DNS History & Zone Transfer Attempts

```bash
# ── DNS HISTORY ─────────────────────────────────────────────────────
# Historical DNS shows old IPs, removed subdomains, infrastructure changes

# SecurityTrails (free tier available):
# https://securitytrails.com/domain/target.com/dns

# Via API:
SECTRAILS_API_KEY="your_key"
curl -s "https://api.securitytrails.com/v1/domain/target.com/history/a" \
  -H "APIKEY: $SECTRAILS_API_KEY" | python3 -m json.tool

# Expected intelligence from DNS history:
# Old IPs → might still run old/vulnerable infrastructure
# Changed nameservers → when they moved to Cloudflare (before: direct IP exposed)
# Removed subdomains → old services that might be restored temporarily

# ViewDNS History:
curl "https://viewdns.info/iphistory/?domain=target.com" 2>/dev/null | \
  grep -oP '(?<=<td>)[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+(?=</td>)'

# ── ZONE TRANSFER ATTEMPT ────────────────────────────────────────────
# Zone transfers (AXFR) should be restricted to trusted secondary nameservers
# Misconfigured DNS = anyone can download the entire zone (all subdomains)

# Find nameservers first:
dig target.com NS +short
# Expected: ns1.target.com, ns2.target.com

# Attempt zone transfer against each nameserver:
dig axfr target.com @ns1.target.com
dig axfr target.com @ns2.target.com

# Expected if VULNERABLE (rare but still found in enterprise):
# target.com.         SOA   ns1.target.com. admin.target.com. ...
# target.com.         A     203.0.113.1
# www.target.com.     A     203.0.113.1
# mail.target.com.    A     203.0.113.2
# dev.target.com.     A     10.0.0.50      ← Internal IP leaked!
# db.target.com.      A     10.0.0.100     ← Database server leaked!
# ... ALL records returned = complete internal map!

# Expected if PROPERLY CONFIGURED:
# Transfer failed. ; Transfer failed.
# (Connection refused or REFUSED)

# Automated check with fierce:
fierce --domain target.com
# Also tries zone transfer + brute forces common subdomains

# dnsx can also check:
echo "target.com" | dnsx -axfr
```

---

# PART 3 — INFRASTRUCTURE INTELLIGENCE

---

## 9. Shodan — The Internet's Search Engine

### Layman's Terms
Shodan scans the entire internet and indexes everything it finds — every open port, every service banner, every SSL certificate, every device. You can search for **specific companies, technologies, vulnerabilities, or device types** and find exposed infrastructure instantly, without ever touching the target systems. It's a search engine for the internet's infrastructure.

```bash
# ── SETUP ────────────────────────────────────────────────────────────
pip3 install shodan
shodan init YOUR_API_KEY  # Free account at shodan.io

# ── SEARCH BY ORGANIZATION ──────────────────────────────────────────
shodan search "org:\"Target Corporation\""
# Expected: all internet-facing devices associated with Target Corp
# Each result shows: IP, open ports, banners, location, OS

shodan search "org:\"Target Corporation\"" --fields ip_str,port,hostnames,data
# Returns structured output

# ── SEARCH BY HOSTNAME/DOMAIN ────────────────────────────────────────
shodan search "hostname:target.com"
shodan search "hostname:.target.com"   # All subdomains

# ── SEARCH BY IP RANGE (from ASN lookup — Section 11) ──────────────
shodan search "net:203.0.113.0/24"
# Expected: all devices in that IP range with their services

# ── SEARCH FOR SPECIFIC TECHNOLOGIES ───────────────────────────────
# Find Apache servers with directory listing:
shodan search "org:\"Target\" Apache"

# Find specific software versions (vulnerability hunting):
shodan search "org:\"Target\" product:\"Microsoft IIS\" version:\"7.5\""
# IIS 7.5 = Server 2008 R2 = potentially end-of-life!

# Find SSL/TLS issues:
shodan search "org:\"Target\" ssl.version:sslv2"
shodan search "org:\"Target\" ssl.version:tlsv1.0"
# Old TLS versions → configuration weakness

# ── HIGH-VALUE SHODAN QUERIES FOR RED TEAM ──────────────────────────
# Default credentials/admin panels:
shodan search "org:\"Target\" title:\"admin\""
shodan search "org:\"Target\" title:\"dashboard\""
shodan search "org:\"Target\" title:\"login\""

# Development/staging systems:
shodan search "org:\"Target\" hostname:dev"
shodan search "org:\"Target\" hostname:staging"
shodan search "org:\"Target\" hostname:test"

# Exposed databases:
shodan search "org:\"Target\" product:MongoDB"
shodan search "org:\"Target\" product:Redis"
shodan search "org:\"Target\" port:5432"   # PostgreSQL
shodan search "org:\"Target\" port:27017"  # MongoDB

# Remote access:
shodan search "org:\"Target\" port:3389"   # RDP
shodan search "org:\"Target\" port:5900"   # VNC
shodan search "org:\"Target\" port:23"     # Telnet

# Industrial/OT:
shodan search "org:\"Target\" port:502"    # Modbus
shodan search "org:\"Target\" port:102"    # Siemens S7

# ── SHODAN HOST LOOKUP ───────────────────────────────────────────────
shodan host 203.0.113.1
# Expected:
# 203.0.113.1
# Hostnames:  web01.target.com
# City:       Seattle
# Country:    United States
# Organization: Target Corporation
# Updated:    2024-01-16T03:14:00
#
# Ports:
# 22/tcp    OpenSSH 8.2p1 Ubuntu
# 80/tcp    nginx 1.18.0
# 443/tcp   nginx 1.18.0 / TLS cert: *.target.com
# 8080/tcp  Apache Tomcat 9.0.45   ← Old Tomcat version!

# ── SHODAN CLI FULL REFERENCE ───────────────────────────────────────
shodan count "org:\"Target Corporation\""  # How many results
shodan download results.json.gz "org:\"Target\"" --limit 1000  # Download
shodan parse results.json.gz --fields ip_str,port,transport  # Parse
shodan myip      # Check your current external IP
shodan info      # Account info and query credits

# ── SHODAN DORKS (PREBUILT SEARCHES) ────────────────────────────────
# All Shodan dorks: https://github.com/jakejarvis/awesome-shodan-queries
# Examples:
shodan search "product:\"Cisco IOS\""           # Cisco routers
shodan search "product:\"Hikvision\" port:554"  # IP cameras
shodan search "\"MongoDB Server Information\" port:27017 -authentication"
# Unauthenticated MongoDB!
shodan search "title:\"Index of /\" http.title:\"Index of /\""
# Open directory listings
```

---

## 10. Censys — Certificate & Infrastructure Search

```bash
# Censys: focuses on certificates, TLS, and structured infrastructure data
# Complementary to Shodan — use both for complete coverage

# Web UI: https://search.censys.io
# Free tier: 250 queries/month

pip3 install censys
# Config: ~/.config/censys/censys.cfg
# [DEFAULT]
# api_id = YOUR_API_ID
# api_secret = YOUR_API_SECRET

# ── SEARCH BY ORGANIZATION ──────────────────────────────────────────
python3 << 'EOF'
from censys.search import CensysHosts

h = CensysHosts()
query = h.search(
    "autonomous_system.name: `Target Corporation`",
    fields=["ip", "services.port", "services.service_name", "services.tls.certificates.leaf_data.subject.common_name"],
    pages=2
)
for host in query:
    print(f"IP: {host['ip']}")
    for svc in host.get('services', []):
        print(f"  Port: {svc['port']} / {svc.get('service_name','')} / {svc.get('tls',{}).get('certificates',{}).get('leaf_data',{}).get('subject',{}).get('common_name','')}")
EOF

# ── SEARCH BY CERTIFICATE ───────────────────────────────────────────
# Find all hosts presenting a certificate for target.com:
python3 << 'EOF'
from censys.search import CensysHosts

h = CensysHosts()
for host in h.search("services.tls.certificates.leaf_data.names: target.com"):
    print(host['ip'])
    for svc in host.get('services', []):
        if 'tls' in svc:
            names = svc.get('tls', {}).get('certificates', {}).get('leaf_data', {}).get('names', [])
            print(f"  Port {svc['port']}: {names}")
EOF
# Expected: all IPs serving certificates that include target.com
# Reveals: CDN origin IPs hiding behind Cloudflare!

# ── FIND ORIGIN IPs BEHIND CDN ──────────────────────────────────────
# Many sites use Cloudflare but their origin server IP leaks via:
# 1. Direct certificate lookup in Censys
# 2. Old DNS history (before they moved to CDN)
# 3. Shodan historical data
# 4. Email server MX records (often not behind CDN)
# 5. DNS lookup of dev/staging (often not behind CDN)

# Find origin by certificate:
python3 << 'EOF'
from censys.search import CensysHosts
h = CensysHosts()
# Look for IPs serving target.com cert but NOT Cloudflare
for host in h.search(
    "services.tls.certificates.leaf_data.names: target.com and NOT autonomous_system.name: CLOUDFLARENET"
):
    print(f"Possible origin: {host['ip']}")
EOF
# If successful: you found the real server IP behind Cloudflare!
# Now you can target the origin directly, bypassing WAF!
```

---

## 11. ASN & IP Range Discovery

```bash
# ASN (Autonomous System Number) = a company's block of IP addresses
# Finding a company's ASN reveals ALL their IP ranges

# ── FIND ASN FROM COMPANY NAME ──────────────────────────────────────
# BGP.he.net (Hurricane Electric):
curl -s "https://bgp.he.net/search?search[search]=target+corporation&commit=Search" 2>/dev/null | \
  grep -oP 'AS\d+' | sort -u
# Expected:
# AS12345   ← Their ASN number

# Or: grep from whois:
whois -h whois.radb.net -- '-i origin AS12345'

# ASN lookup via API:
curl -s "https://api.bgpview.io/search?query_term=Target+Corporation" | \
  python3 -c "import json,sys; data=json.load(sys.stdin); [print(a['asn'],a['name']) for a in data.get('data',{}).get('asns',[])]"

# ── GET IP RANGES FROM ASN ──────────────────────────────────────────
# Once you have ASN number:
whois -h whois.radb.net -- '-i origin AS12345' | grep "route:"
# Expected:
# route: 203.0.113.0/24
# route: 198.51.100.0/22
# route: 198.0.0.0/16

# Via BGP.he.net:
curl -s "https://bgp.he.net/AS12345#_prefixes" 2>/dev/null | \
  grep -oP '\d+\.\d+\.\d+\.\d+/\d+' | sort -u

# ── AMASS FOR ASN ────────────────────────────────────────────────────
amass intel -asn 12345
# Returns: all IP ranges for that ASN, related domains

# ── BUILD SCOPE FROM IP RANGES ─────────────────────────────────────
# All discovered IP ranges = full scope for your engagement
# Feed into nmap/masscan for infrastructure discovery:
echo "203.0.113.0/24" > target_ranges.txt
echo "198.51.100.0/22" >> target_ranges.txt

sudo masscan -iL target_ranges.txt -p0-65535 --rate 10000 -oG masscan_results.txt
# Fast scan of all ports on all company IPs
```

---

## 12. Cloud Infrastructure Fingerprinting

```bash
# Most companies use cloud → identify which cloud and how

# ── IDENTIFY CLOUD PROVIDER FROM IP ────────────────────────────────
# Check if IP is in AWS, Azure, or GCP ranges:
python3 << 'EOF'
import json, urllib.request

# AWS IP ranges:
aws_url = "https://ip-ranges.amazonaws.com/ip-ranges.json"
with urllib.request.urlopen(aws_url) as r:
    aws = json.load(r)

target_ip = "203.0.113.1"
import ipaddress
target = ipaddress.ip_address(target_ip)

for prefix in aws['prefixes']:
    network = ipaddress.ip_network(prefix['ip_prefix'])
    if target in network:
        print(f"AWS: {prefix['region']} ({prefix['service']})")
        break
EOF

# Azure ranges:
# https://www.microsoft.com/en-us/download/details.aspx?id=56519

# GCP ranges:
curl -s "https://www.gstatic.com/ipranges/cloud.json" | \
  python3 -c "import json,sys; d=json.load(sys.stdin); [print(p['ipv4Prefix'],p.get('service',''),p.get('scope','')) for p in d.get('prefixes',[]) if 'ipv4Prefix' in p]" | grep "203.0.113"

# ── IDENTIFY S3 BUCKETS ──────────────────────────────────────────────
# Target may have publicly accessible S3 buckets with sensitive data!
# Common naming patterns:
COMPANY="target"
for pattern in "$COMPANY" "${COMPANY}-backup" "${COMPANY}-dev" \
  "${COMPANY}-prod" "${COMPANY}-staging" "${COMPANY}-data" \
  "${COMPANY}-files" "${COMPANY}-assets" "${COMPANY}-static" \
  "${COMPANY}-media" "${COMPANY}-uploads" "${COMPANY}-logs"; do
  
  # Check if bucket exists:
  status=$(curl -s -o /dev/null -w "%{http_code}" \
    "https://${pattern}.s3.amazonaws.com/" 2>/dev/null)
  
  case $status in
    200) echo "[PUBLIC] https://${pattern}.s3.amazonaws.com/";;
    403) echo "[EXISTS-PRIVATE] ${pattern}.s3.amazonaws.com";;
    301) echo "[REDIRECT] ${pattern}.s3.amazonaws.com";;
  esac
done

# ── S3 BUCKET CONTENT IF PUBLIC ────────────────────────────────────
# If bucket returns 200 (public):
aws s3 ls s3://target-backup/ --no-sign-request
# Expected if accessible:
# 2024-01-01 03:14:00  104857600 database_backup_2024.sql.gz
# 2024-01-01 03:14:00  52428800  config_backup.tar.gz
# These files may contain: database dumps, credentials, source code!

# ── CLOUD METADATA LEAKAGE ──────────────────────────────────────────
# If you find a web app on cloud infrastructure:
# SSRF to metadata endpoint → see Cloud Pentesting module
# 169.254.169.254 (AWS IMDSv1 — covered in Cloud module)
```

---

# PART 4 — PEOPLE & ORGANIZATIONAL INTELLIGENCE

---

## 14. Employee Enumeration — LinkedIn, GitHub, OSINT

```bash
# ── LINKEDIN EMPLOYEE DISCOVERY ──────────────────────────────────────
# LinkedIn reveals: who works there, their roles, tech stack, org structure

# Manual: Search LinkedIn for "Target Corporation" employees
# Look for: security team (understand their defenses)
#           IT/infrastructure (understand their tech stack)
#           developers (find GitHub accounts)
#           executives (highest-value phishing targets)

# ── LINKEDIN2USERNAME (GENERATE USERNAME LIST) ──────────────────────
git clone https://github.com/initstring/linkedin2username
python3 linkedin2username.py -c "Target Corporation" -n 1000
# Generates usernames in all common corporate formats:
# johndoe           j.doe
# john.doe          doe.john
# doej              jdoe
# john_doe          johndo     (truncated)

# Output usable for:
# → Password spraying (Active Directory module)
# → Email generation (combine with email format)
# → Phishing target list

# ── GITHUB EMPLOYEE DISCOVERY ────────────────────────────────────────
# Employees often link their employer on GitHub
# Organization GitHub accounts show all members

# Search GitHub for company employees:
# https://github.com/orgs/target-corp/members (if org is public)
# Search: site:github.com "Target Corporation" in:profile

# Tool: github-dorker / gitrob
gitrob analyze target-corp
# Expected: finds all employee accounts, their repos, and any sensitive files

# ── USERNAME CROSS-PLATFORM SEARCH ──────────────────────────────────
# Find a username across ALL social platforms:
python3 -m pip install sherlock
sherlock john.doe
# Checks: GitHub, Twitter, Instagram, Reddit, LinkedIn, 200+ sites
# Expected:
# [+] GitHub: https://github.com/john.doe
# [+] Twitter: https://twitter.com/johndoe_
# [+] LinkedIn: https://linkedin.com/in/johndoe
# [+] Stack Overflow: https://stackoverflow.com/users/johndoe

# ── EXTRACT EMAILS FROM GITHUB ──────────────────────────────────────
# GitHub commits contain author email addresses!
git clone https://github.com/target-corp/some-repo
cd some-repo
git log --format="%ae %an" | sort -u | grep "@target.com"
# Expected:
# alice@target.com  Alice Smith
# bob@target.com    Bob Johnson
# → Confirms email format: firstname@target.com
# → Adds real names to employee list
```

---

## 15. Email Address Discovery & Harvesting

```bash
# ── HUNTER.IO (BEST COMMERCIAL SOURCE) ───────────────────────────────
# hunter.io finds emails associated with a domain
# Free: 25 searches/month, Paid: unlimited

# Web UI: https://hunter.io/domain-search
# Enter: target.com → returns verified emails

# API:
HUNTER_KEY="your_api_key"
curl "https://api.hunter.io/v2/domain-search?domain=target.com&api_key=$HUNTER_KEY" | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
emails = data.get('data', {}).get('emails', [])
print(f'Found {len(emails)} emails')
for e in emails:
    print(f'{e[\"value\"]} ({e[\"type\"]}) [{e.get(\"confidence\",\"?\")}%] {e.get(\"first_name\",\"\")} {e.get(\"last_name\",\"\")}')
"
# Expected:
# Found 23 emails
# alice.smith@target.com (personal) [95%] Alice Smith   ← IT Director
# security@target.com (generic) [80%]
# support@target.com (generic) [75%]

# ── THEHARVESTER ────────────────────────────────────────────────────
# theHarvester queries many sources simultaneously:
theHarvester -d target.com -b all -l 500 -f harvester_results
# -d = domain
# -b all = all sources (Google, LinkedIn, Shodan, etc.)
# -l 500 = limit 500 results
# -f = save to file

# Expected:
# [*] Emails found: 15
# alice@target.com
# bob.johnson@target.com
# hr@target.com
# [*] Hosts found: 45
# mail.target.com:203.0.113.2
# dev.target.com:10.0.0.50
# [*] IPs found: 12
# 203.0.113.1, 203.0.113.2...

# ── GOOGLE DORK FOR EMAILS ──────────────────────────────────────────
# Google search for email addresses from target domain:
# site:target.com "@target.com" filetype:pdf
# "@target.com" site:linkedin.com
# "@target.com" site:github.com

# ── EMAILREP.IO — EMAIL REPUTATION ──────────────────────────────────
curl "https://emailrep.io/alice@target.com"
# Expected:
# {
#   "email": "alice@target.com",
#   "reputation": "high",
#   "suspicious": false,
#   "references": 3,
#   "details": {
#     "blacklisted": false,
#     "malicious_activity": false,
#     "data_breach": true,    ← This email appeared in a breach!
#     "domain_exists": true,
#     "domain_reputation": "high",
#     "profiles": ["linkedin", "github"]  ← Has these accounts!
#   }
# }
```

---

## 16. Email Format Identification

```bash
# CRITICAL: Before phishing or spraying, know the email format
# Wrong format = bounce, tip-off, wasted effort

# ── COMMON FORMATS ──────────────────────────────────────────────────
# firstname.lastname@company.com         (most common)
# firstlast@company.com                  (initials+last)
# firstname@company.com                  (first only)
# f.lastname@company.com                 (initial.last)
# flastname@company.com                  (initial+last)
# firstname_lastname@company.com         (underscore)

# ── IDENTIFY FORMAT FROM FOUND EMAILS ──────────────────────────────
# Use theHarvester, Hunter.io, LinkedIn output to find real emails
# Pattern:
# alice.smith@target.com → format: firstname.lastname
# asmith@target.com → format: firstinitiallastname

# Tool: emailfinder
pip3 install emailfinder
emailfinder -d target.com
# Tries to verify format by submitting to login forms

# ── VERIFY EMAILS WITHOUT SENDING ──────────────────────────────────
# SMTP verification (check if email exists):
python3 << 'EOF'
import smtplib, dns.resolver

def verify_email(email):
    domain = email.split('@')[1]
    # Get MX records
    mx_records = dns.resolver.resolve(domain, 'MX')
    mx_host = str(sorted(mx_records, key=lambda x: x.preference)[0].exchange)
    
    try:
        smtp = smtplib.SMTP(timeout=10)
        smtp.connect(mx_host, 25)
        smtp.helo('verify.local')
        smtp.mail('verify@verify.local')
        code, msg = smtp.rcpt(email)
        smtp.quit()
        return code == 250  # 250 = valid, 550 = invalid
    except Exception as e:
        return None  # Can't determine

result = verify_email("alice@target.com")
print("Valid" if result else "Invalid or couldn't verify")
EOF
# Note: Many servers don't respond accurately to prevent enumeration
# This is semi-active (sends packets to mail server)

# ── FORMAT-BASED LIST GENERATION ────────────────────────────────────
python3 << 'EOF'
names = [
    ("Alice", "Smith"),
    ("Bob", "Johnson"),
    ("Carol", "Williams")
]
domain = "target.com"
format_type = "firstname.lastname"  # Change based on identified format

for first, last in names:
    if format_type == "firstname.lastname":
        print(f"{first.lower()}.{last.lower()}@{domain}")
    elif format_type == "firstlast":
        print(f"{first[0].lower()}{last.lower()}@{domain}")
    elif format_type == "firstname":
        print(f"{first.lower()}@{domain}")
EOF
```

---

# PART 5 — CREDENTIAL & DATA LEAK INTELLIGENCE

---

## 18. Breach Database Searching

### Layman's Terms
Billions of credentials have been leaked in breaches over the years. Many employees **reuse passwords** between work and personal accounts. If their personal email/password appears in a breach, there's a good chance that same password (or a slight variation) works for their work account. Breach data also reveals security questions, phone numbers, and password patterns.

```bash
# ── HAVE I BEEN PWNED (HIBP) ────────────────────────────────────────
# The most complete breach database

# Check one email:
curl -s "https://haveibeenpwned.com/api/v3/breachedaccount/alice@target.com" \
  -H "hibp-api-key: YOUR_API_KEY" | python3 -m json.tool
# Expected:
# [{"Name":"Adobe","Title":"Adobe","Domain":"adobe.com","BreachDate":"2013-10-04",...},
#  {"Name":"LinkedIn","Title":"LinkedIn",...},
#  {"Name":"Dropbox","Title":"Dropbox",...}]
# → alice@target.com appeared in 3 major breaches!

# Check domain for all compromised emails:
curl -s "https://haveibeenpwned.com/api/v3/breachedaccount/alice@target.com?domain=target.com" \
  -H "hibp-api-key: YOUR_API_KEY"

# ── DEHASHED — SEARCH FOR ACTUAL PASSWORDS ──────────────────────────
# DeHashed provides the actual plaintext passwords from breaches (paid)
# https://dehashed.com

# Search by domain (shows all breached accounts at company):
curl "https://api.dehashed.com/search?query=email:@target.com" \
  -H "Authorization: Basic $(echo -n 'email:api_key' | base64)"

# Expected:
# {"id":123,"email":"alice@target.com","username":"alice_work","password":"Password123!","hashed_password":"$2a$...","name":"Alice Smith"}
# → Real credentials from breach!

# ── SEARCHING BREACH DATA ────────────────────────────────────────────
# Several resources for breach data search:
# IntelligenceX: https://intelx.io (searches Pastebin, dumps, darkweb)
# Leak-Lookup: https://leak-lookup.com
# Snusbase: https://snusbase.com

# ── PASSWORD PATTERN ANALYSIS ────────────────────────────────────────
# Even if you find hashed passwords — analyze the patterns
# When employees change passwords, they often follow patterns:
# Summer2019! → Summer2020! → Summer2021!
# Password1! → Password2! → Password3!

# Generate password variations for spraying:
python3 << 'EOF'
base_passwords = ["Password1!", "Summer2019!", "Company123!"]
variations = []
years = ["2022", "2023", "2024"]
seasons = ["Spring", "Summer", "Fall", "Winter"]

for pwd in base_passwords:
    # Try with different years
    for year in years:
        variations.append(pwd.replace("2019", year))
        variations.append(pwd.replace("2020", year))

# Common password patterns for corporate environments:
corp_patterns = [
    "Welcome1!", "Welcome123!", "Password1!", "Password123!",
    "Summer2024!", "Winter2024!", "Spring2024!", "Fall2024!",
    "January2024!", "February2024!"
]
variations.extend(corp_patterns)

for v in sorted(set(variations)):
    print(v)
EOF
```

---

## 19. Paste Site Monitoring & History

```bash
# Paste sites (Pastebin, Ghostbin, etc.) sometimes contain:
# - Leaked credentials
# - Internal documentation
# - Source code with embedded secrets
# - Configuration files

# ── SEARCH GOOGLE FOR PASTE LEAKS ──────────────────────────────────
# Google dorks targeting paste sites:
# site:pastebin.com "target.com" password
# site:pastebin.com "@target.com"
# site:pastebin.com "target corporation"

# ── PASTEBIN SEARCH API ─────────────────────────────────────────────
# Pastebin API (scraping search):
curl "https://psbdmp.ws/api/v3/search/target.com" 2>/dev/null | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
for p in data.get('data', []):
    print(f\"https://pastebin.com/{p['id']} - {p.get('date','')}\")
"

# ── INTELLIGENCE X ──────────────────────────────────────────────────
# IntelligenceX searches: Pastebin, GitHub, darkweb, etc.
# API: https://intelx.io/api

# ── PWNBIN ──────────────────────────────────────────────────────────
# Search multiple paste sites simultaneously:
pip3 install pwnbin
pwnbin -k "target.com password"
```

---

## 20. GitHub Secret Scanning — Public Repos

### Layman's Terms
Developers frequently **accidentally commit secrets** to public GitHub repos — API keys, passwords, database connection strings, private certificates. Once committed, secrets remain in git history even after deletion. Many of these secrets are still valid years later because nobody realized they were exposed.

```bash
# ── TRUFFLEHOG — GIT SECRET SCANNER ────────────────────────────────
pip3 install truffleHog
# Or: go install github.com/trufflesecurity/trufflehog/v3@latest

# Scan a specific repository:
trufflehog git https://github.com/target-corp/webapp
# Expected:
# 🐷🔑🐷  TruffleHog. Unearths Secrets. 🐷🔑🐷
# Found verified result 🔑
# Detector Type: AWS
# Raw result: AKIAIOSFODNN7EXAMPLE
# File: src/config/aws_config.py
# Line: 15
# Commit: abc123def456 (3 years ago)  ← Still in history!

# Scan all repos for an organization:
trufflehog github --org=target-corp --token=GITHUB_TOKEN
# Scans ALL public repos of the organization!

# ── GITROB ─────────────────────────────────────────────────────────
# Find sensitive files in GitHub repos:
git clone https://github.com/michenriksen/gitrob
cd gitrob && go build
./gitrob target-corp
# Looks for: .env, *.pem, *.key, id_rsa, *.config, credentials.*

# ── GITLEAKS ────────────────────────────────────────────────────────
pip3 install gitleaks
# Scan entire git history:
gitleaks detect --source=/path/to/cloned/repo --log-opts="--all"

# ── GITHUB DORKS ────────────────────────────────────────────────────
# Search GitHub via search operators:

# Find API keys:
# site:github.com "target.com" "api_key"
# site:github.com "target.com" password
# site:github.com "target.com" secret

# Find SSH private keys:
# site:github.com "BEGIN RSA PRIVATE KEY" "target"
# site:github.com "BEGIN EC PRIVATE KEY" "target"

# Find database credentials:
# site:github.com "target.com" "jdbc:mysql"
# site:github.com "target.com" "mongodb://"
# site:github.com "target.com" connectionstring

# Find AWS keys for target:
# site:github.com org:target-corp AKIAIOSFODNN7

# ── AUTOMATED GITHUB SEARCH ─────────────────────────────────────────
python3 << 'EOF'
import requests

GITHUB_TOKEN = "ghp_your_token"
headers = {
    "Authorization": f"token {GITHUB_TOKEN}",
    "Accept": "application/vnd.github.v3+json"
}

# Search for company secrets:
queries = [
    'target.com password',
    'target.com secret_key',
    'target.com aws_access',
    '"@target.com" api_key',
]

for query in queries:
    url = f"https://api.github.com/search/code?q={query}&per_page=10"
    r = requests.get(url, headers=headers)
    data = r.json()
    items = data.get('items', [])
    if items:
        print(f"\n=== '{query}' ===")
        for item in items[:5]:
            print(f"  {item['repository']['full_name']}: {item['path']}")
            print(f"  {item['html_url']}")
EOF
```

---

## 21. Google Dorks — Advanced Search Operators

```bash
# Google dorks use search operators to find sensitive information indexed by Google
# All of this is completely passive — you're just using Google

# ── FUNDAMENTAL OPERATORS ──────────────────────────────────────────
# site:target.com          → only results from target.com
# filetype:pdf             → only PDF files
# intitle:"keyword"        → keyword in page title
# inurl:"keyword"          → keyword in URL
# intext:"keyword"         → keyword in page body
# -keyword                 → exclude results with keyword
# "exact phrase"           → exact phrase match
# OR                       → either term

# ── FILE EXPOSURE ───────────────────────────────────────────────────
# Exposed documents:
site:target.com filetype:pdf
site:target.com filetype:xlsx "confidential"
site:target.com filetype:doc "internal"
site:target.com filetype:xls "password"

# Exposed configuration files:
site:target.com filetype:conf
site:target.com filetype:cfg
site:target.com filetype:env
site:target.com filetype:yml "password"
site:target.com "wp-config.php"

# Exposed backup files:
site:target.com filetype:bak
site:target.com filetype:sql
site:target.com filetype:log "error"

# ── ADMIN & LOGIN PORTALS ────────────────────────────────────────────
site:target.com inurl:admin
site:target.com inurl:login
site:target.com inurl:/wp-admin
site:target.com inurl:dashboard
site:target.com intitle:"admin panel"
site:target.com intitle:"login" inurl:admin

# ── ERROR MESSAGES & DEBUG ───────────────────────────────────────────
site:target.com "error" "stack trace"
site:target.com "ORA-01756"     ← Oracle SQL error (reveals database type)
site:target.com "MySQL error"
site:target.com "Warning: mysql"
site:target.com intitle:"index of" "parent directory"   ← Directory listing!

# ── SENSITIVE CONTENT ────────────────────────────────────────────────
site:target.com "confidential" -filetype:pdf
site:target.com "not for public" OR "internal use only"
site:target.com "api_key" OR "apikey" OR "api key"
site:target.com "secret" "password"

# ── SUBDOMAINS DISCOVERY ────────────────────────────────────────────
site:*.target.com -site:www.target.com -site:mail.target.com
# Lists all indexed subdomains except the ones you already know

# ── TECHNOLOGY IDENTIFICATION ────────────────────────────────────────
site:target.com "Powered by WordPress"
site:target.com "Powered by Drupal"
site:target.com inurl:joomla
site:target.com "Proudly powered by"

# ── EMAIL EXPOSURE ───────────────────────────────────────────────────
site:target.com "mailto:" "@target.com"
"@target.com" site:linkedin.com
"@target.com" site:github.com

# ── COMPLETE DORK SEQUENCE FOR A PENTEST ───────────────────────────
# Run these systematically and document each result:
DOMAIN="target.com"
cat << 'EOF' > google_dorks.txt
site:${DOMAIN} filetype:pdf intitle:"confidential"
site:${DOMAIN} filetype:xls "password"
site:${DOMAIN} filetype:env
site:${DOMAIN} inurl:admin
site:${DOMAIN} inurl:login inurl:admin
site:${DOMAIN} intitle:"index of"
site:${DOMAIN} "stack trace" "error"
site:*.${DOMAIN} -site:www.${DOMAIN}
"@${DOMAIN}" site:github.com
"@${DOMAIN}" site:pastebin.com
site:${DOMAIN} "api_key" OR "secret_key"
site:${DOMAIN} "Welcome to" inurl:phpmyadmin
EOF
echo "Open each dork in browser:"
while IFS= read -r dork; do
    encoded=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$dork'))")
    echo "https://www.google.com/search?q=$encoded"
done < google_dorks.txt
```

---

# PART 6 — SPECIALIZED OSINT TOOLS

---

## 22. theHarvester — Automated Email & Infrastructure

```bash
# theHarvester is built into Kali and queries dozens of sources simultaneously

# ── FULL SCAN ────────────────────────────────────────────────────────
theHarvester \
  -d target.com \
  -b all \
  -l 500 \
  -f harvester_output
# -d = target domain
# -b all = use all data sources
# -l 500 = limit 500 results
# -f = output file (creates .html and .json)

# ── SPECIFIC SOURCES ─────────────────────────────────────────────────
# Most useful sources:
theHarvester -d target.com -b google,bing,linkedin,shodan,hunter

# Available sources:
# anubis, baidu, bevigil, binaryedge, bing, bingapi, bufferoverun,
# censys, certspotter, crtsh, dnsdumpster, duckduckgo, fullhunt,
# github-code, google, hackertarget, hunter, intelx,
# linkedin, linkedin_links, otx, pentesttools, projectdiscovery,
# qwant, rapiddns, rocketreach, securityTrails, shodan,
# sublist3r, threatminer, trello, urlscan, virustotal, yahoo

# ── PARSE RESULTS ────────────────────────────────────────────────────
# Results saved to: harvester_output.json
python3 << 'EOF'
import json

with open('harvester_output.json') as f:
    data = json.load(f)

print("=== EMAILS ===")
for email in data.get('emails', []):
    print(f"  {email}")

print("\n=== HOSTS ===")
for host in data.get('hosts', []):
    print(f"  {host}")

print("\n=== IPs ===")
for ip in data.get('ips', []):
    print(f"  {ip}")

print("\n=== INTERESTING URLS ===")
for url in data.get('interesting_urls', []):
    print(f"  {url}")
EOF
```

---

## 23. recon-ng — Modular OSINT Framework

```bash
# recon-ng: Python framework with modules for OSINT
# Think: Metasploit but for OSINT

recon-ng

# ── SETUP WORKSPACE ──────────────────────────────────────────────────
[recon-ng][default] > workspaces create target_corp
[recon-ng][target_corp] > workspaces select target_corp

# ── ADD SEEDS ────────────────────────────────────────────────────────
[recon-ng][target_corp] > db insert domains
domain (TEXT): target.com
notes (TEXT): main domain
# Adds to database

# ── INSTALL MODULES ──────────────────────────────────────────────────
[recon-ng][target_corp] > marketplace install all
# OR specific modules:
[recon-ng][target_corp] > marketplace install recon/domains-hosts/hackertarget
[recon-ng][target_corp] > marketplace install recon/domains-hosts/certificate_transparency
[recon-ng][target_corp] > marketplace install recon/hosts-hosts/resolve
[recon-ng][target_corp] > marketplace install recon/domains-contacts/whois_pocs

# ── RUN MODULES ──────────────────────────────────────────────────────
# Find subdomains via certificate transparency:
[recon-ng][target_corp] > modules load recon/domains-hosts/certificate_transparency
[recon-ng][target_corp][certificate_transparency] > run
# Expected: populates hosts table with all CT subdomains

# Find contacts via WHOIS:
[recon-ng][target_corp] > modules load recon/domains-contacts/whois_pocs
[recon-ng][target_corp][whois_pocs] > run
# Expected: populates contacts table

# Resolve all discovered hosts:
[recon-ng][target_corp] > modules load recon/hosts-hosts/resolve
[recon-ng][target_corp][resolve] > run
# Expected: resolves all hosts to IPs

# ── VIEW RESULTS ─────────────────────────────────────────────────────
[recon-ng][target_corp] > show hosts
# Expected:
# +------------------+---------------+-------+
# | host             | ip_address    | notes |
# +------------------+---------------+-------+
# | dev.target.com   | 10.0.0.50     |       |
# | api.target.com   | 203.0.113.100 |       |
# | mail.target.com  | 203.0.113.2   |       |

[recon-ng][target_corp] > show contacts
# Expected:
# +--------------+---------------------+-------+
# | name         | email               | notes |
# +--------------+---------------------+-------+
# | Alice Smith  | alice@target.com    |       |
# | Bob Johnson  | bob@target.com      |       |

# ── GENERATE REPORT ──────────────────────────────────────────────────
[recon-ng][target_corp] > modules load reporting/html
[recon-ng][target_corp][html] > options set FILENAME /tmp/recon_report.html
[recon-ng][target_corp][html] > options set CREATOR "Red Team"
[recon-ng][target_corp][html] > options set CUSTOMER "Target Corporation"
[recon-ng][target_corp][html] > run
# Creates comprehensive HTML report with all findings
```

---

## 24. Maltego — Visual Link Analysis

```bash
# Maltego: visual intelligence platform for mapping relationships
# Shows connections between: people, emails, domains, IPs, social media
# Free version (Maltego CE): limited transforms
# Paid: full transform set

# Download: https://www.maltego.com/downloads/

# ── KEY MALTEGO CONCEPTS ─────────────────────────────────────────────
# Entity: any data point (person, email, domain, IP, company)
# Transform: action that discovers related entities
# Graph: visual map of all discovered entities and their connections

# ── TYPICAL MALTEGO WORKFLOW ─────────────────────────────────────────
# 1. Start with domain entity: target.com
# 2. Run "To DNS Name [Passive DNS]" → discovers subdomains
# 3. Run "To IP Address [DNS]" → resolves subdomains to IPs
# 4. Run "To Website [Quick Lookup]" → web servers
# 5. Run "To Email Address [PGP Email Search]" → employee emails
# 6. Click email → run "To Person [Full Contact]" → LinkedIn profiles
# 7. Click person → run "To Social Networks" → social accounts
# 8. Visual graph shows everything connected

# ── MALTEGO FREE TRANSFORMS ────────────────────────────────────────
# DNS to IP (DNS lookup)
# IP to domains (reverse DNS)
# Domain to whois info (registrant email)
# Email to person (email lookup)
# Company to employees (LinkedIn)
# URL to links (web scraping)
# IP to location (geolocation)

# ── EXPORT RESULTS ────────────────────────────────────────────────────
# Maltego → Export → CSV or Excel
# Use for: building phishing target list, documenting attack surface

# ── ALTERNATIVE: VISUALIZE WITH GEPHI ────────────────────────────────
# Export recon-ng data to Gephi format for visualization:
[recon-ng][target_corp] > modules load reporting/gexf
[recon-ng][target_corp][gexf] > run
# Then open in Gephi for visual analysis
```

---

## 25. SpiderFoot — Automated OSINT

```bash
# SpiderFoot: automated OSINT tool that queries 200+ data sources
# Runs continuously, following leads automatically

sudo apt install spiderfoot
# Or: pip3 install spiderfoot

# Start SpiderFoot web UI:
python3 sf.py -l 127.0.0.1:5001
# Open: http://127.0.0.1:5001

# ── COMMAND LINE ─────────────────────────────────────────────────────
# Scan a target domain:
python3 sf.py -s target.com \
  -t INTERNET_NAME \
  -m sfp_dnsresolve,sfp_ssl,sfp_shodan,sfp_hunter,sfp_whois,sfp_subdomain \
  -o json > spiderfoot_results.json

# ── USEFUL MODULES ───────────────────────────────────────────────────
# sfp_dnsresolve     = DNS resolution
# sfp_ssl            = SSL certificate analysis
# sfp_shodan         = Shodan lookup
# sfp_hunter         = Hunter.io email search
# sfp_github         = GitHub search
# sfp_crt            = Certificate transparency
# sfp_whois          = WHOIS lookup
# sfp_passivedns     = Passive DNS
# sfp_haveibeenpwned = Breach checking
# sfp_linkedin       = LinkedIn search
# sfp_twitter        = Twitter search

# Run full passive scan:
python3 sf.py -s target.com -t INTERNET_NAME --modules all -o json > full_results.json

# ── PARSE RESULTS ────────────────────────────────────────────────────
cat spiderfoot_results.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
# Group by type:
from collections import defaultdict
by_type = defaultdict(list)
for r in data:
    by_type[r.get('type','unknown')].append(r.get('data',''))
for t, items in sorted(by_type.items()):
    print(f'\n=== {t} ({len(items)}) ===')
    for i in items[:5]:
        print(f'  {i}')
"
```

---

## 26. Amass — Comprehensive Attack Surface Mapping

```bash
# Amass: most comprehensive open-source attack surface mapping tool
# Combines passive + active + brute force subdomain enumeration

sudo apt install amass
# Or: go install -v github.com/owasp-amass/amass/v4/...@master

# ── PASSIVE ONLY ──────────────────────────────────────────────────────
amass enum -passive -d target.com -o amass_passive.txt
# Queries: certificates, DNS datasets, APIs, web archives
# No DNS queries to target's nameservers

# ── ACTIVE (DNS brute force) ─────────────────────────────────────────
amass enum -active -d target.com -o amass_active.txt -brute
# -active = sends DNS queries
# -brute = wordlist brute force

# ── WITH API KEYS ────────────────────────────────────────────────────
# Amass uses free APIs but many require keys for full results
# Config file: ~/.config/amass/config.ini
cat > ~/.config/amass/config.ini << 'EOF'
[data_sources.Shodan]
[data_sources.Shodan.Credentials]
apikey = YOUR_SHODAN_KEY

[data_sources.SecurityTrails]
[data_sources.SecurityTrails.Credentials]
apikey = YOUR_SECTRAILS_KEY

[data_sources.Censys]
[data_sources.Censys.Credentials]
apiid = YOUR_CENSYS_ID
apisecret = YOUR_CENSYS_SECRET
EOF

# Run with API keys:
amass enum -passive -d target.com -config ~/.config/amass/config.ini -o amass_full.txt

# ── INTEL MODE (ASN/ORG) ─────────────────────────────────────────────
# Find all domains associated with an ASN:
amass intel -asn 12345 -o asn_domains.txt

# Find domains from company name:
amass intel -org "Target Corporation" -o org_domains.txt

# ── VISUALIZATION ────────────────────────────────────────────────────
# View Amass database:
amass db -names
# All discovered names

amass db -show -d target.com
# All data for a domain

# Export to D3.js visualization:
amass viz -d3 -d target.com -o amass_viz.html
# Opens interactive network graph in browser!
```

---

# PART 7 — SOCIAL MEDIA & WEB INTELLIGENCE

---

## 27. LinkedIn Intelligence

```bash
# LinkedIn is the richest source of corporate intelligence

# ── WHAT LINKEDIN REVEALS ────────────────────────────────────────────
# Without any LinkedIn account:
# - Company employee count
# - Department sizes
# - Recent hires (new employees = easier targets, less security-aware)
# - Technology used (from job postings)
# - Org structure (from titles + management)

# With a LinkedIn account (research profile):
# - Full employee list (name, title, location)
# - Work history (understand relationships)
# - Education (common password patterns: UniversityName2010)
# - Skills (understand what tech they use)
# - Recent posts (current projects, conference talks)

# ── LINKEDIN SEARCH TECHNIQUES ──────────────────────────────────────
# Find employees: LinkedIn → Search → People → Current company: Target Corp
# Filter: By department (IT, Security, Engineering)
# Filter: By seniority (C-suite, Director, Manager)
# Export search results: LinkedIn Recruiter (paid)

# ── MANUAL OSINT FROM LINKEDIN ─────────────────────────────────────
# 1. Note all security team members
#    → Understanding security posture (what tools they use)
#    → Their conference talks reveal internal systems
#
# 2. Note IT infrastructure team
#    → Job titles reveal tech stack
#    → Certifications reveal vendor products (Cisco cert = Cisco network)
#
# 3. Note recent hires
#    → New employees from where? (if from a specific competitor → they brought tools)
#
# 4. Note recent departures (if visible)
#    → Employees who left angry may leak info
#
# 5. Note executive assistants
#    → Often targeted in phishing (access to executive calendars/email)

# ── LINKEDIN2USERNAME ─────────────────────────────────────────────────
# Already covered in Section 14:
python3 linkedin2username.py -c "Target Corporation" -n 1000
# Use output for:
# - Password spraying combined with email format
# - Phishing target list
# - AD user validation
```

---

## 29. Wayback Machine & Web Archives

```bash
# Wayback Machine (web.archive.org) stores historical snapshots of websites
# Reveals: old pages, removed content, historical emails, old tech, past breaches

# ── MANUAL USE ───────────────────────────────────────────────────────
# Visit: https://web.archive.org/web/*/target.com
# Browse historical versions of any page

# ── API ACCESS ────────────────────────────────────────────────────────
# Find all archived URLs for a domain:
curl "http://web.archive.org/cdx/search/cdx?url=*.target.com&output=json&fl=original,timestamp&collapse=urlkey" 2>/dev/null | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
urls = set()
for item in data[1:]:  # Skip header
    urls.add(item[0])
print(f'Total unique URLs archived: {len(urls)}')
for url in sorted(urls)[:50]:
    print(url)
" 2>/dev/null

# ── WAYBACKURLS ──────────────────────────────────────────────────────
# Tool to extract all archived URLs:
go install github.com/tomnomnom/waybackurls@latest
echo "target.com" | waybackurls
# Expected: hundreds/thousands of archived URLs
# Look for:
# - /admin, /backup, /.env, /config (sensitive paths)
# - Old API endpoints (may still work)
# - Old login forms (different tech, potential vulns)
# - Removed pages with sensitive data

# ── EXTRACT PARAMETERS FOR FUZZING ──────────────────────────────────
# Old URLs reveal parameter names → test on current site:
echo "target.com" | waybackurls | grep "?" | \
  sed 's/=.*/=/' | sort -u > parameters.txt
# Gives you all historical URL parameters
# Use for: XSS testing, SQLi testing on current app

# ── HISTORICAL DIRECTORY LISTINGS ────────────────────────────────────
echo "target.com" | waybackurls | grep "index of\|directory listing" 2>/dev/null
# May find archived directory listings with sensitive files

# ── JAVASCRIPT FILES IN HISTORY ──────────────────────────────────────
echo "target.com" | waybackurls | grep "\.js$" | sort -u
# Old JS files may contain: API keys, endpoints, internal URLs
for js_url in $(echo "target.com" | waybackurls | grep "\.js$" | head -20); do
    curl -s "$js_url" 2>/dev/null | grep -iE "api[_-]?key|secret|token|password" | head -3
done
```

---

## 30. Job Listings as Intel Source

```bash
# Job postings are GOLD for understanding a company's technology and security posture

# ── WHAT JOB LISTINGS REVEAL ────────────────────────────────────────
# Security Engineer posting:
#   "Experience with CrowdStrike Falcon, Splunk SIEM, Palo Alto NGFW"
#   → EDR: CrowdStrike (Defender evasion module becomes more specific)
#   → SIEM: Splunk (understand what gets logged)
#   → Firewall: Palo Alto (understand network controls)
#
# Cloud Engineer posting:
#   "AWS with Terraform, Kubernetes EKS, Helm charts"
#   → Cloud: AWS
#   → Orchestration: Kubernetes
#   → IaC: Terraform (look for exposed Terraform state files!)
#
# Developer posting:
#   "Java Spring Boot microservices, PostgreSQL, Redis, RabbitMQ"
#   → Backend tech stack revealed
#   → Database type confirmed
#   → Message queue type (potential SSRF target)
#
# SRE posting:
#   "Grafana, Prometheus, PagerDuty"
#   → Monitoring stack (look for exposed Grafana!)
#   → Alerting tool
#
# Network Engineer posting:
#   "Cisco ASA, Juniper SRX, BGP/OSPF"
#   → Network vendor (check Shodan for Cisco-specific vulnerabilities)

# ── AUTOMATED JOB SCRAPING ──────────────────────────────────────────
python3 << 'EOF'
import requests
from bs4 import BeautifulSoup

# LinkedIn Jobs (public API):
url = "https://www.linkedin.com/jobs/search/?keywords=Target+Corporation+security"
headers = {"User-Agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36"}
# Note: LinkedIn requires auth for full access; scrape public job listings from other sources

# Indeed (public search):
indeed_url = "https://www.indeed.com/jobs?q=Target+Corporation+it+security&l="
r = requests.get(indeed_url, headers=headers)
soup = BeautifulSoup(r.text, 'html.parser')

# Extract job titles and descriptions:
for job in soup.find_all('div', class_='job_seen_beacon')[:5]:
    title = job.find('span', {'title': True})
    if title:
        print(f"Job: {title['title']}")
EOF

# ── KEYWORDS TO GREP IN JOB LISTINGS ────────────────────────────────
# After collecting job listing text:
LISTINGS_FILE="job_listings.txt"
echo "=== Security Tools ==="
grep -iE "crowdstrike|carbon black|defender|cylance|sentinelone|darktrace" "$LISTINGS_FILE"
echo "=== SIEM ==="
grep -iE "splunk|qradar|sentinel|elastic|arcsight" "$LISTINGS_FILE"
echo "=== Cloud ==="
grep -iE "aws|azure|gcp|google cloud|amazon" "$LISTINGS_FILE"
echo "=== Tech Stack ==="
grep -iE "java|python|ruby|golang|nodejs|react|angular|spring|django" "$LISTINGS_FILE"
echo "=== Databases ==="
grep -iE "mysql|postgres|oracle|mssql|mongodb|redis|elasticsearch" "$LISTINGS_FILE"
```

---

# PART 8 — BUILDING THE TARGET PROFILE

---

## 31. Aggregating Intel — Building the Complete Picture

```bash
# ══════════════════════════════════════════════════════════════════
# COMPLETE OSINT WORKFLOW — EXAMPLE RUN
# Target: Example Corp (target.com)
# ══════════════════════════════════════════════════════════════════

TARGET="target.com"
OUTPUT_DIR="osint_target_$(date +%Y%m%d)"
mkdir -p "$OUTPUT_DIR"/{domains,people,infrastructure,credentials,leaks}

echo "╔═══════════════════════════════════════════╗"
echo "║  OSINT Collection: $TARGET                "
echo "╚═══════════════════════════════════════════╝"

# ── PHASE 1: DOMAIN INTELLIGENCE ────────────────────────────────────
echo "[*] Phase 1: Domain Intelligence"

# WHOIS
whois "$TARGET" > "$OUTPUT_DIR/domains/whois.txt"
echo "  [+] WHOIS data collected"

# DNS Records
for type in A AAAA MX NS TXT SOA; do
    dig "$TARGET" "$type" +short >> "$OUTPUT_DIR/domains/dns_records.txt"
done
echo "  [+] DNS records collected"

# DMARC check
dig "_dmarc.$TARGET" TXT +short >> "$OUTPUT_DIR/domains/dmarc.txt"
DMARC=$(cat "$OUTPUT_DIR/domains/dmarc.txt")
if echo "$DMARC" | grep -q "p=none"; then
    echo "  [!] DMARC p=none — Domain can be spoofed for phishing!"
fi

# Subdomains (passive)
subfinder -d "$TARGET" -all -o "$OUTPUT_DIR/domains/subdomains_passive.txt" 2>/dev/null
echo "  [+] $(wc -l < "$OUTPUT_DIR/domains/subdomains_passive.txt") passive subdomains found"

# Certificate Transparency
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" 2>/dev/null | \
  python3 -c "
import json,sys
data=json.load(sys.stdin)
names=set()
for c in data:
    for n in c.get('name_value','').split('\n'):
        n=n.strip().lstrip('*.')
        if '.' in n: names.add(n)
print('\n'.join(sorted(names)))" > "$OUTPUT_DIR/domains/ct_subdomains.txt"
echo "  [+] $(wc -l < "$OUTPUT_DIR/domains/ct_subdomains.txt") CT subdomains found"

# Merge and resolve
cat "$OUTPUT_DIR/domains/subdomains_passive.txt" \
    "$OUTPUT_DIR/domains/ct_subdomains.txt" | \
    sort -u | dnsx -r 8.8.8.8 -o "$OUTPUT_DIR/domains/live_subdomains.txt" 2>/dev/null
echo "  [+] $(wc -l < "$OUTPUT_DIR/domains/live_subdomains.txt") live subdomains"

# ── PHASE 2: INFRASTRUCTURE ─────────────────────────────────────────
echo "[*] Phase 2: Infrastructure Intelligence"

# Shodan
shodan search "hostname:$TARGET" --fields ip_str,port,hostnames,data \
  > "$OUTPUT_DIR/infrastructure/shodan.txt" 2>/dev/null
echo "  [+] Shodan data collected"

# ASN lookup
whois -h whois.radb.net -- "-i origin $(dig +short "$TARGET" | head -1 | xargs whois | grep 'origin:' | awk '{print $2}')" \
  > "$OUTPUT_DIR/infrastructure/asn.txt" 2>/dev/null

# Web technologies
cat "$OUTPUT_DIR/domains/live_subdomains.txt" | \
  httpx -title -status-code -tech-detect -o "$OUTPUT_DIR/infrastructure/web_services.txt" 2>/dev/null
echo "  [+] Web services fingerprinted"

# ── PHASE 3: PEOPLE ─────────────────────────────────────────────────
echo "[*] Phase 3: People Intelligence"

# Email harvesting
theHarvester -d "$TARGET" -b all -l 200 -f "$OUTPUT_DIR/people/theharvester" 2>/dev/null
echo "  [+] Email harvesting complete"

# ── PHASE 4: CREDENTIAL LEAKS ───────────────────────────────────────
echo "[*] Phase 4: Credential & Leak Intelligence"

# GitHub scanning
trufflehog github --org="${TARGET%%.*}" > "$OUTPUT_DIR/leaks/github_secrets.txt" 2>/dev/null
echo "  [+] GitHub secret scan complete"

# ── PHASE 5: GENERATE SUMMARY ────────────────────────────────────────
cat > "$OUTPUT_DIR/SUMMARY.md" << SUMMARY
# OSINT Summary: $TARGET
Generated: $(date)

## Domains & Subdomains
- Live subdomains found: $(wc -l < "$OUTPUT_DIR/domains/live_subdomains.txt")
- Unique subdomains (passive): $(cat "$OUTPUT_DIR/domains/subdomains_passive.txt" "$OUTPUT_DIR/domains/ct_subdomains.txt" | sort -u | wc -l)

## High-Value Subdomains Found
$(grep -iE "dev|staging|admin|vpn|api|gitlab|jenkins|jira|sso" "$OUTPUT_DIR/domains/live_subdomains.txt" | head -20)

## Email Infrastructure
$(cat "$OUTPUT_DIR/domains/dns_records.txt" | grep -i "MX\|mail")

## DMARC Status
$(cat "$OUTPUT_DIR/domains/dmarc.txt")

## Web Services Summary
$(grep "\[200\]" "$OUTPUT_DIR/infrastructure/web_services.txt" | head -20)

## Emails Found
$(grep "@$TARGET" "$OUTPUT_DIR/people/theharvester.json" 2>/dev/null | head -20)

SUMMARY

echo "[+] OSINT collection complete!"
echo "[+] Results in: $OUTPUT_DIR/"
echo "[+] Summary: $OUTPUT_DIR/SUMMARY.md"
```

---

## 32. Attack Surface Map from OSINT

```
TRANSLATING OSINT INTO ATTACK PATHS:

FROM DNS RECORDS:
  MX → mail.target.com:203.0.113.2
  → Test: Outlook Web App / Webmail (credential brute force)
  → Test: Email security bypass (phishing infrastructure)
  
  TXT v=spf1 include:mailchimp.net
  → Uses MailChimp → check for misconfigured mailing lists
  
  DMARC p=none
  → Domain can be spoofed → phishing opportunity!

FROM SUBDOMAINS:
  dev.target.com → 10.0.0.50 (internal IP leaked!)
  → Development server, likely less security, more debug
  
  jenkins.target.com → [200] Jenkins v2.319
  → CI/CD exposed → look for unauth access, old Jenkins RCEs
  
  staging.target.com → [200] Same app as prod but without WAF?
  → Test same payloads that prod WAF blocks
  
  vpn.target.com → [200] Cisco AnyConnect
  → VPN endpoint → credential spray with employee list
  
  old.target.com → [200] WordPress 4.7.2
  → Old WordPress version → WordPress REST API exploit!

FROM SHODAN:
  203.0.113.1 → Port 8080 Tomcat 7.0.73
  → Old Tomcat → likely CVE-2017-12617 (JSP upload)
  
  203.0.113.2 → Port 27017 MongoDB (no auth banner)
  → Unauthenticated MongoDB → dump all data!
  
  203.0.113.5 → Port 5900 VNC (no auth)
  → Direct desktop access!

FROM JOB LISTINGS:
  "Experience with CrowdStrike Falcon"
  → EDR is CrowdStrike → tailor AV evasion to bypass Falcon
  
  "Jenkins, GitLab CI, Kubernetes"
  → Infrastructure stack known → targeted enumeration

FROM BREACH DATA:
  alice@target.com found in Adobe breach (2013)
  → Try breach password against: VPN, webmail, Citrix
  → Try variations: breach_password1!, breach_password@2024

FROM GITHUB:
  AWS API key found in old commit: AKIAIOSFODNN7EXAMPLE
  → Test if key still valid: aws sts get-caller-identity
  → Enumerate AWS permissions: aws iam list-user-policies
```

---

## 33. OSINT-Driven Attack Planning

```
OSINT → ATTACK PLAN TEMPLATE:

Based on OSINT, plan attacks in order of likely success and impact:

PRIORITY 1 — QUICK WINS (highest probability):
  □ Breach credentials tested against VPN (vpn.target.com)
  □ Breach credentials tested against webmail (owa.target.com)
  □ Jenkins unauthenticated access check (jenkins.target.com)
  □ MongoDB unauthenticated (203.0.113.2:27017)
  □ Default creds on exposed Tomcat manager (203.0.113.1:8080/manager)

PRIORITY 2 — CREDENTIAL ATTACKS (high impact):
  □ Password spray against VPN using employee list + common passwords
  □ Password spray against webmail (OWA) with lockout awareness
  □ Phishing campaign using DMARC p=none + spoofed domain
    (alice@target.com receives email from target.com — no SPF rejection)

PRIORITY 3 — VULNERABILITY EXPLOITATION:
  □ Tomcat 7.0.73 → CVE-2017-12617 (JSP file upload)
  □ WordPress 4.7.2 on old.target.com → REST API privilege escalation
  □ Jenkins version-specific CVE check

PRIORITY 4 — CONTINUED RECONNAISSANCE (if needed):
  □ Social engineering using discovered employee intel
  □ Deeper subdomain brute force
  □ AWS key privilege escalation

TOOLS FOR EACH PRIORITY:
  VPN/OWA spray: crackmapexec, Ruler, SprayingToolkit
  Jenkins: metasploit auxiliary/scanner/jenkins/jenkins_login
  MongoDB: mongodump (dump all data)
  Tomcat: msfvenom WAR file + deploy via manager
  WordPress: wpscan + exploit module

OSINT → PHISHING PREPARATION:
  To: alice@target.com, bob.johnson@target.com (from theHarvester)
  From: Spoofed target.com address (DMARC p=none allows this!)
  Subject: Urgent: Your VPN access expires today
  Body: References real VPN URL (vpn.target.com found in OSINT)
  Payload: Credential harvest page mimicking real VPN portal
  
  Intelligence makes phishing 10x more convincing:
  - Real employee names (LinkedIn)
  - Real internal system names (subdomains)
  - Real VPN URL (OSINT)
  - Real email format (confirmed from harvester)
  - IT team member name as sender (LinkedIn)
```

---

*Next module: **API Security Testing** — REST API attacks, GraphQL exploitation, JWT vulnerabilities, IDOR chains, mass assignment, rate limiting bypass, and full API pentest methodology.*

*Cross-references:*
- *Subdomain takeover (after finding subdomains): `Web_Application_Security_RedTeam_Field_Manual.md`*
- *Credential use after breach data: `Active_Directory_RedTeam_Field_Manual.md` (password spraying)*
- *GitHub secrets → AWS access: `Cloud_Networking_Sections_18_to_36.md`*
- *Wireless enumeration: `Wireless_Security_RedTeam_Field_Manual.md` (passive recon)*

*Tools: amass, subfinder, dnsx, httpx, theHarvester, recon-ng, Maltego, SpiderFoot,*
*Shodan CLI, trufflehog, gitrob, gitleaks, sherlock, linkedin2username,*
*waybackurls, censys, whois, dig, curl*

*Resources: osintframework.com, hunter.io, dehashed.com, haveibeenpwned.com,*
*crt.sh, shodan.io, censys.io, securitytrails.com, wigle.net*

*Labs: TraceLabs OSINT CTF (real missing persons cases, legal practice),*
*OSINT Dojo (practice challenges), HackTheBox OSINT challenges,*
*PentesterLab OSINT exercises*