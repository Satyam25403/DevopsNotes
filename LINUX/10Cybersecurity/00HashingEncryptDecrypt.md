# Hashing, Encryption, and Decryption

Complete guide to cryptographic operations, hashing algorithms, encryption methods, and practical security tools.

## Table of Contents
- [Hashing](#hashing)
- [Encryption vs Encoding](#encryption-vs-encoding)
- [Symmetric Encryption](#symmetric-encryption)
- [Asymmetric Encryption](#asymmetric-encryption)
- [SSL/TLS Certificates](#ssltls-certificates)
- [Password Hashing](#password-hashing)
- [File Encryption](#file-encryption)

---

## Hashing

### What is Hashing?

**One-way function that converts data to fixed-size output.**

**Characteristics:**
- ✅ Deterministic (same input = same output)
- ✅ Fixed output size
- ✅ Fast computation
- ✅ One-way (cannot reverse)
- ✅ Avalanche effect (small change = completely different hash)

**Visual:**
```
Input (any size)
      ↓
  [Hash Function]
      ↓
Output (fixed size)

Example:
"hello"           → a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e
"hello world"     → b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9
"hello worlD"     → 9b71d224bd62f3785d96d46ad3ea3d73319bfbc2890caadae2dff72519673ca7

Note: Even one character change produces completely different hash
```

### Common Hash Algorithms

**MD5 (128-bit):**
```bash
echo -n "hello" | md5sum
# Output: 5d41402abc4b2a76b9719d911017c592

# Hash file
md5sum filename.txt
```

**SHA-1 (160-bit):**
```bash
echo -n "hello" | sha1sum
# Output: aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d
```

**SHA-256 (256-bit) - Recommended:**
```bash
echo -n "hello" | sha256sum
# Output: 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824

# Hash file
sha256sum filename.txt

# Verify file integrity
sha256sum -c checksum.txt
```

**SHA-512 (512-bit):**
```bash
echo -n "hello" | sha512sum
```

### Use Cases

**File Integrity Verification:**
```bash
# Create checksum file
sha256sum file.iso > file.iso.sha256

# Verify later
sha256sum -c file.iso.sha256
```

**Password Storage:**
```bash
# Never store plain passwords!
# Store hash instead:
echo -n "mypassword" | sha256sum
# Store: a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3

# Verification:
# Hash user input and compare with stored hash
```

**Git Commits:**
```
Each commit has SHA-1 hash:
commit abc1234def5678...
```

### Hash Comparison

**Visual:**
```
Algorithm  Output Size  Speed    Security Status
─────────────────────────────────────────────────
MD5        128-bit      Fast     ❌ Broken (collisions found)
SHA-1      160-bit      Fast     ⚠️  Deprecated (collisions found)
SHA-256    256-bit      Medium   ✅ Secure (current standard)
SHA-512    512-bit      Medium   ✅ Secure
SHA-3      Variable     Medium   ✅ Secure (newest standard)
BLAKE2     256/512-bit  Fastest  ✅ Secure

Recommendation: Use SHA-256 or SHA-512
```

---

## Encryption vs Encoding

### Key Differences

**Visual:**
```
┌─────────────┬──────────────┬──────────────┐
│   Hashing   │  Encryption  │   Encoding   │
├─────────────┼──────────────┼──────────────┤
│ One-way     │ Two-way      │ Two-way      │
│ Fixed size  │ Variable     │ Variable     │
│ No key      │ Needs key    │ No key       │
│ Irreversible│ Reversible   │ Reversible   │
│             │              │              │
│ Use: Verify │ Use: Protect │ Use: Format  │
│ integrity   │ confidential │ conversion   │
└─────────────┴──────────────┴──────────────┘

Examples:
Hashing:     hello → 2cf24dba5fb0a30e...
Encryption:  hello → U2FsdGVkX1... (with key)
Encoding:    hello → aGVsbG8= (base64)
```

### Base64 Encoding

**Not encryption! Just formatting.**

```bash
# Encode
echo "hello" | base64
# Output: aGVsbG8K

# Decode
echo "aGVsbG8K" | base64 -d
# Output: hello

# Encode file
base64 file.txt > file.b64

# Decode file
base64 -d file.b64 > file.txt
```

**Use cases:**
- Email attachments
- Data URLs
- Binary data in text format
- NOT for security!

---

## Symmetric Encryption

### AES (Advanced Encryption Standard)

**Same key for encryption and decryption.**

**Visual:**
```
Encryption:
Plaintext + Key → [AES Encrypt] → Ciphertext

Decryption:
Ciphertext + Key → [AES Decrypt] → Plaintext

Key must be shared securely!
```

### OpenSSL AES Encryption

**Encrypt file:**
```bash
# AES-256-CBC encryption
openssl enc -aes-256-cbc -salt -in file.txt -out file.enc

# Enter password when prompted
# -salt adds random salt for security
```

**Decrypt file:**
```bash
openssl enc -aes-256-cbc -d -in file.enc -out file.txt
# Enter same password
```

**Encrypt with password in command:**
```bash
# Not recommended (password in history)
openssl enc -aes-256-cbc -salt -in file.txt -out file.enc -k mypassword

# Better: Use password file
echo "mypassword" > pass.txt
chmod 600 pass.txt
openssl enc -aes-256-cbc -salt -in file.txt -out file.enc -pass file:pass.txt
```

### GPG (GNU Privacy Guard)

**Symmetric encryption:**
```bash
# Encrypt file
gpg -c file.txt
# Creates: file.txt.gpg

# Decrypt file
gpg file.txt.gpg
# Creates: file.txt
```

**With specific cipher:**
```bash
gpg --cipher-algo AES256 -c file.txt
```

---

## Asymmetric Encryption

### RSA Key Pairs

**Different keys for encryption and decryption.**

**Visual:**
```
Key Pair Generation:
┌──────────────┐
│ Generate Keys│
└──────┬───────┘
       ├─→ Public Key  (share with everyone)
       └─→ Private Key (keep secret)

Encryption Flow:
Sender                    Receiver
  │                          │
  │─ Get Public Key ────────→│
  │                          │
  │  [Encrypt with Public]   │
  │                          │
  │─ Send Ciphertext ───────→│
  │                          │
  │                [Decrypt with Private]
  │                          │

Public Key:  Lock (anyone can lock)
Private Key: Key (only you can unlock)
```

### Generate RSA Keys

**Using OpenSSL:**
```bash
# Generate private key (4096-bit)
openssl genrsa -out private.pem 4096

# Extract public key
openssl rsa -in private.pem -pubout -out public.pem

# View private key
openssl rsa -in private.pem -text -noout

# View public key
openssl rsa -pubin -in public.pem -text -noout
```

**Using ssh-keygen:**
```bash
# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -C "user@example.com"
# Creates: ~/.ssh/id_rsa (private) and ~/.ssh/id_rsa.pub (public)

# Ed25519 (modern, faster)
ssh-keygen -t ed25519 -C "user@example.com"
```

### Encrypt with Public Key

```bash
# Encrypt file with public key
openssl rsautl -encrypt -pubin -inkey public.pem -in file.txt -out file.enc

# Decrypt with private key
openssl rsautl -decrypt -inkey private.pem -in file.enc -out file.txt
```

### GPG Asymmetric Encryption

**Generate key pair:**
```bash
# Generate GPG key
gpg --full-generate-key

# List keys
gpg --list-keys

# Export public key
gpg --export -a "Your Name" > public.key

# Import someone's public key
gpg --import their_public.key
```

**Encrypt for recipient:**
```bash
# Encrypt file for recipient
gpg --encrypt --recipient "Recipient Name" file.txt
# Creates: file.txt.gpg

# Decrypt (recipient uses their private key)
gpg --decrypt file.txt.gpg > file.txt
```

---

## SSL/TLS Certificates

### Generate Self-Signed Certificate

```bash
# Generate private key and certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout private.key \
  -out certificate.crt

# Interactive prompts for:
# - Country
# - State
# - City
# - Organization
# - Common Name (domain)
```

**View certificate:**
```bash
openssl x509 -in certificate.crt -text -noout
```

### Certificate Signing Request (CSR)

**For production certificates:**
```bash
# Generate private key
openssl genrsa -out domain.key 2048

# Generate CSR
openssl req -new -key domain.key -out domain.csr

# Submit CSR to Certificate Authority (CA)
# CA returns signed certificate
```

### Let's Encrypt (Free SSL)

```bash
# Install certbot
sudo apt install certbot

# Generate certificate
sudo certbot certonly --standalone -d example.com

# Certificates stored in:
# /etc/letsencrypt/live/example.com/
#   - fullchain.pem (certificate)
#   - privkey.pem (private key)

# Auto-renewal
sudo certbot renew
```

---

## Password Hashing

### bcrypt (Recommended)

**Why bcrypt:**
- Slow by design (prevents brute force)
- Includes salt automatically
- Adaptive (can increase work factor)

**Python example:**
```python
import bcrypt

# Hash password
password = b"mypassword"
salt = bcrypt.gensalt()
hashed = bcrypt.hashpw(password, salt)
print(hashed)
# $2b$12$KIXxLVE8XKXyHF.yZKZ5mO7x9Y...

# Verify password
if bcrypt.checkpw(password, hashed):
    print("Password correct!")
```

### argon2 (Modern Standard)

**Winner of Password Hashing Competition.**

**Python example:**
```python
from argon2 import PasswordHasher

ph = PasswordHasher()

# Hash password
hash = ph.hash("mypassword")
print(hash)

# Verify
try:
    ph.verify(hash, "mypassword")
    print("Password correct!")
except:
    print("Wrong password!")
```

### PBKDF2

```bash
# Using OpenSSL
openssl passwd -6 -salt xyz mypassword
# Uses SHA-512 with salt
```

---

## File Encryption

### Encrypt Directory

**Using tar and GPG:**
```bash
# Compress and encrypt
tar czf - directory/ | gpg -c > backup.tar.gz.gpg

# Decrypt and extract
gpg -d backup.tar.gz.gpg | tar xzf -
```

### Encrypted Archives

**7zip with password:**
```bash
# Install 7zip
sudo apt install p7zip-full

# Create encrypted archive
7z a -p"password" -mhe=on archive.7z files/
# -p: password
# -mhe=on: encrypt headers (hide filenames)

# Extract
7z x archive.7z
```

### Full Disk Encryption

**LUKS (Linux Unified Key Setup):**
```bash
# Create encrypted volume
cryptsetup luksFormat /dev/sdX

# Open encrypted volume
cryptsetup luksOpen /dev/sdX myvolume

# Format and mount
mkfs.ext4 /dev/mapper/myvolume
mount /dev/mapper/myvolume /mnt

# Close when done
umount /mnt
cryptsetup luksClose myvolume
```

---

## Quick Reference

### Hashing Commands

```bash
md5sum file.txt           # MD5 hash
sha1sum file.txt          # SHA-1 hash
sha256sum file.txt        # SHA-256 hash
sha512sum file.txt        # SHA-512 hash
```

### Encryption Commands

```bash
# Symmetric (OpenSSL)
openssl enc -aes-256-cbc -salt -in file.txt -out file.enc
openssl enc -aes-256-cbc -d -in file.enc -out file.txt

# Asymmetric (GPG)
gpg -c file.txt                    # Symmetric
gpg --encrypt --recipient "Name"   # Asymmetric
gpg --decrypt file.txt.gpg
```

### Key Generation

```bash
# SSH keys
ssh-keygen -t rsa -b 4096
ssh-keygen -t ed25519

# OpenSSL keys
openssl genrsa -out private.pem 4096
openssl rsa -in private.pem -pubout -out public.pem

# GPG keys
gpg --full-generate-key
```

### Best Practices

```
✅ DO:
- Use SHA-256 or better for hashing
- Use AES-256 for symmetric encryption
- Use RSA-4096 or Ed25519 for asymmetric
- Use bcrypt or argon2 for passwords
- Keep private keys secure (chmod 600)
- Use strong passwords/passphrases
- Enable 2FA when available

❌ DON'T:
- Use MD5 or SHA-1 for security
- Store passwords in plaintext
- Share private keys
- Use weak passwords
- Hardcode passwords in code
- Use encryption without authentication
```

---

This guide covers hashing, encryption, decryption methods and cryptographic best practices for Linux systems.