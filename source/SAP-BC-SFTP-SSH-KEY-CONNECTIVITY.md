---
Category: Procedure
Command: sftp, ssh-keygen, sshpass, grep, cat, date
Date: 2026-04-28
DocType: How-to guide
Environment: Development, Quality, Production
Filesystem: ~/.ssh, /usr/sap, /etc, /etc/ssh
OS: Linux
Priority: Very high
Product: OpenSSH, RHEL, SAP Netweaver
Protocol: SFTP, SSH
Scope: Unix Administration, SAP Administration
Server: SAP-PRD, SAP-QUA, SAP-DEV, VENDOR-SFTP-PRD, VENDOR-SFTP-UAT
Tag: ed25519, key pair, migration
Team: SAP Basis
Topic: Connectivity, Migration, Security, Troubleshooting
---

# SFTP Connectivity - SSH Key Authentication

## Overview

This document describes how to establish and validate SFTP connectivity from the SAP Payroll
systems to the vendor SFTP server using SSH key-based authentication. It covers key generation,
testing procedures, and known outcomes across environments (Development/DEV, Quality/QUA,
Production/PRD), with emphasis on the transition from password-based authentication to the
recommended ED25519 key method.

> **NOTE:** Password-based SFTP authentication via `sshpass` is **obsolete** and must not be
> used in production. It is documented here only for historical reference and as a baseline
> for initial connectivity tests.

---

## Prerequisites

- SSH client (`OpenSSH 8.0+`) installed on the SAP host.
- The SAP OS user for the target environment (`devadm`, `quaadm`, or `prdadm`) must have
  write access to `~/.ssh/`.
- The vendor must have registered and activated the public key on their SFTP server before
  key-based authentication can succeed.
- The vendor SFTP server hostkey must be present in `~/.ssh/known_hosts`
  (auto-added on first successful connection, or added manually).
---

## Environments and Connection Parameters

| Environment | SAP System | SAP Host | OS User | SFTP Server | SFTP Username |
|---|---|---|---|---|---|
| Development | DEV | SAP-DEV | devadm | VENDOR-SFTP-UAT | ftpuser-uat@vendor-domain |
| Quality | QUA | SAP-QUA | quaadm | VENDOR-SFTP-UAT | ftpuser-uat@vendor-domain |
| Production | PRD | SAP-PRD | prdadm | VENDOR-SFTP-PRD | ftpuser-prd@vendor-domain |

---

## Key Pairs Reference

| Environment | Algorithm | Key file (private) | Fingerprint |
|---|---|---|---|
| UAT (RSA) | RSA 4096 | `~/.ssh/id_acme-ssh_key-uat` | `SHA256:QLVAYE8vilDQKM56FU0Tho8S4C+MaKpAXqthlnM1yCA` |
| PRD (RSA) | RSA 4096 | `~/.ssh/id_acme-ssh_key-prd` | `SHA256:Nfo2Pw8+AZQBnkmWvjexW82uQCFUC0cywU+c9lALhq0` |
| UAT (ED25519) | ED25519 256 | `~/.ssh/ssh_key_uat` | `SHA256:Z6iykBXNvtP84njBNILg1p3LZrxnGOhmwQ6+IYF/kDA` |
| PRD (ED25519) | ED25519 256 | `~/.ssh/id_acme-ssh_key-prod` | `SHA256:TYcA2jIghuLWBiClrq1bqStoMAdfeH8UwPWc0DSm1QM` |

> **NOTE:** Vendor SFTP server host key fingerprint (RSA):
> `SHA256:LZUIVXeQXBtUqpcCGNRYzqrL5Jr5TB7gjgFza6Mm9gA`

---

## Procedure

### Step 1 - Generate a new SSH key pair (ED25519, recommended)

ED25519 is the algorithm recommended by the Unix/infrastructure team. It provides stronger
security than RSA with a shorter key length, and is supported by OpenSSH 8.0+.

```bash
ssh-keygen -t ed25519 -a 100
```

Example session (on SAP-DEV as devadm):

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/devadm/.ssh/id_ed25519): /home/devadm/.ssh/ssh_key_uat
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/devadm/.ssh/ssh_key_uat.
Your public key has been saved in /home/devadm/.ssh/ssh_key_uat.pub.
The key fingerprint is:
SHA256:Z6iykBXNvtP84njBNILg1p3LZrxnGOhmwQ6+IYF/kDA devadm@SAP-DEV
```

Verify the generated key:

```bash
ssh-keygen -lf ~/.ssh/ssh_key_uat.pub
```

Expected output:

```
256 SHA256:Z6iykBXNvtP84njBNILg1p3LZrxnGOhmwQ6+IYF/kDA devadm@SAP-DEV (ED25519)
```

> **NOTE:** Use `-a 100` (key derivation rounds) to strengthen the key against brute-force
> attacks on the passphrase, if a passphrase is set.

---

### Step 2 - Send the public key to the vendor

After generation, send the **public key file** (`*.pub`) to the vendor contact, and request
it be registered on the SFTP server for the corresponding environment and username.

Do not proceed to connection testing until the vendor confirms the key has been activated.

---

### Step 3 - Verify key fingerprints match

Before and after key deployment, verify that both private and public key fingerprints are identical:

```bash
# Private key fingerprint
ssh-keygen -lf ~/.ssh/id_acme-ssh_key-prod

# Public key fingerprint
ssh-keygen -lf ~/.ssh/id_acme-ssh_key-prod.pub
```

Both commands must return the **same SHA256 fingerprint**. A mismatch indicates a key file
integrity problem.

---

### Step 4 - Create a connection test script

Create a minimal test script for each environment. The `-v` flag enables verbose debug output,
which is essential for diagnosing authentication failures.

**UAT test script** (run as `devadm` on SAP-DEV):

```bash
#!/usr/bin/bash
FTPUSR=ftpuser-uat@vendor-domain
FTPSRV=VENDOR-SFTP-UAT
FTPKEY="$HOME/.ssh/ssh_key_uat"
sftp -v -i $FTPKEY $FTPUSR@$FTPSRV
```

**PRD test script** (run as `prdadm` on SAP-PRD):

```bash
#!/usr/bin/bash
FTPUSR=ftpuser-prd@vendor-domain
FTPSRV=VENDOR-SFTP-PRD
FTPKEY="$HOME/.ssh/id_acme-ssh_key-prod"
sftp -v -i $FTPKEY $FTPUSR@$FTPSRV
```

Run the test with a timestamp to record when connectivity was validated:

```bash
date && ./test_ssh_key.sh
```

---

### Step 5 - Interpret the verbose output

#### Successful key-based authentication

The following lines in the verbose output confirm success:

```
debug1: Offering public key: <keyfile> <algorithm> <fingerprint> explicit
debug1: Server accepts key: <keyfile> <algorithm> <fingerprint> explicit
debug1: Authentication succeeded (publickey).
Authenticated to VENDOR-SFTP-PRD (IP-PRD:22).
Connected to VENDOR-SFTP-PRD.
```

Full successful session example (RSA PRD key, SAP-PRD):

```
debug1: Will attempt key: /home/prdadm/.ssh/id_acme-ssh_key-prd RSA SHA256:Nfo2Pw8+AZQBnkmWvjexW82uQCFUC0cywU+c9lALhq0 explicit
debug1: Offering public key: /home/prdadm/.ssh/id_acme-ssh_key-prd RSA SHA256:Nfo2Pw8+AZQBnkmWvjexW82uQCFUC0cywU+c9lALhq0 explicit
debug1: Server accepts key: /home/prdadm/.ssh/id_acme-ssh_key-prd RSA SHA256:Nfo2Pw8+AZQBnkmWvjexW82uQCFUC0cywU+c9lALhq0 explicit
debug1: Authentication succeeded (publickey).
Authenticated to VENDOR-SFTP-PRD (IP-PRD:22).
Connected to VENDOR-SFTP-PRD.
```

#### Failed key-based authentication (key not yet registered by vendor)

If the vendor has not yet registered the public key, the server will decline it and fall through
to keyboard-interactive (password prompt):

```
debug1: Offering public key: /home/prdadm/.ssh/id_acme-ssh_key-prod ED25519 SHA256:TYcA2jIghuLWBiClrq1bqStoMAdfeH8UwPWc0DSm1QM explicit
debug1: Authentications that can continue: password,keyboard-interactive,publickey
debug1: Next authentication method: keyboard-interactive
Password authentication
Password:
```

This is **not** an SSH client error - the client offered the key correctly. The server
simply rejected it, meaning the public key is not yet registered or is registered under a
different username. **Do not enter a password.** Interrupt the session (`Ctrl+C`) and
follow up with the vendor.

---

### Step 6 - (Historical) Password-based authentication via sshpass

> **WARNING:** This method is **obsolete** and **not recommended** for production use.
> It is retained here for historical reference only.

Initial connectivity tests were performed using `sshpass` before SSH key pairs were exchanged.

```bash
#!/usr/bin/bash
FTPUSR=ftpuser-uat@vendor-domain
FTPSRV=VENDOR-SFTP-UAT
FTPPAS=XXXXXXXXXXXXXXXXXXXX
LOG=$HOME/sftp-test.log

echo "$(date '+%d.%m.%Y - %H:%M:%S') => Attempting connection" >> $LOG
sshpass -p $FTPPAS sftp -v $FTPUSR@$FTPSRV << EOF >> $LOG 2>&1
EOF
```

Successful output (filtered):

```
Password authentication
Authenticated to VENDOR-SFTP-UAT (IP-UAT:22).
Connected to VENDOR-SFTP-UAT.
```

---

## Verification

After a successful connection, confirm the following from the verbose output:

| Check | Expected value |
|---|---|
| KEX algorithm | `ecdh-sha2-nistp256` |
| Host key algorithm | `rsa-sha2-512` |
| Cipher (both directions) | `aes256-ctr` |
| MAC | `hmac-sha2-256-etm@openssh.com` |
| Authentication method | `publickey` |
| Server host key fingerprint | `SHA256:LZUIVXeQXBtUqpcCGNRYzqrL5Jr5TB7gjgFza6Mm9gA` |

---

## Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| Server declines key, prompts for password | Public key not yet registered by vendor | Contact vendor; provide correct public key and username |
| `Host key verification failed` | `known_hosts` entry changed or missing | Clear stale entry: `ssh-keygen -R VENDOR-SFTP-PRD`; reconnect to re-add |
| `no match: APACHE-SSHD-2.5.1` | Debug message about unknown server banner | Informational only; does not affect connectivity |
| `Warning: Permanently added the RSA host key for IP address` | IP added to `known_hosts` after hostname already present | Normal behaviour; not an error |
| Fingerprint mismatch between private and public key | Wrong key file pair | Re-run `ssh-keygen -lf` on both files; regenerate if needed |

---

## Key Algorithm Comparison

| Algorithm | Key size | `ssh-keygen` command | Notes |
|---|---|---|---|
| RSA | 3072–4096 bit | `ssh-keygen -t rsa -b 4096` | Widely supported; larger key size required |
| ED25519 | 256 bit (fixed) | `ssh-keygen -t ed25519 -a 100` | **Recommended**; modern, compact, strong |

> ED25519 is the standard going forward.
