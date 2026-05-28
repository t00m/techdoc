---
Category: Procedure
Command: scp, ssh, rsync
Date: 2021-02-08
DocType: Reference
OS: Linux, Unix
Product: AIX, Solaris, SSH, RHEL, Ubuntu, SFTP
Protocol: SSH
Scope: Unix Administration
Status: Released
Tag: copy, remote, transfer
Topic: Operation, Security
---

# Copy files between servers via SCP

## Overview

`scp` (Secure Copy Protocol) copies files and directories between hosts over SSH.
It uses SSH for authentication and encryption, so no extra setup is needed beyond
an existing SSH configuration.

General syntax:

```bash
scp [options] <source> <destination>
```

Remote paths use the format `user@host:/path`.

---

## Copy a local file to a remote server

```bash
scp /local/path/file.txt user@host:/remote/path/
```

---

## Copy a remote file to the local machine

```bash
scp user@host:/remote/path/file.txt /local/path/
```

---

## Copy between two remote servers

```bash
scp user1@host1:/path/file.txt user2@host2:/path/
```

---

## Copy a directory recursively

Use `-r` to copy a full directory tree.

```bash
scp -r /local/dir user@host:/remote/path/
```

---

## Use a specific SSH private key

```bash
scp -i ~/.ssh/id_rsa /local/file.txt user@host:/remote/path/
```

---

## Use a non-standard SSH port

```bash
scp -P 2222 /local/file.txt user@host:/remote/path/
```

> **NOTE:** The port flag for `scp` is uppercase `-P`, unlike `ssh` which uses lowercase `-p`.

---

## Preserve file metadata (timestamps, permissions)

```bash
scp -p /local/file.txt user@host:/remote/path/
```

---

## Limit bandwidth usage

Value is in Kbits/s.

```bash
scp -l 1024 /local/file.txt user@host:/remote/path/
```

---

## Copy with verbose output (useful for debugging)

```bash
scp -v /local/file.txt user@host:/remote/path/
```

---

## Common options summary

| Option | Description |
|--------|-------------|
| `-r` | Recursive copy (directories) |
| `-P <port>` | Non-standard SSH port |
| `-i <key>` | Private key file |
| `-p` | Preserve file timestamps and permissions |
| `-l <kbps>` | Limit transfer bandwidth |
| `-v` | Verbose mode |
| `-C` | Enable compression |
| `-q` | Quiet mode (suppress progress) |

---

## Notes

- `scp` relies on the remote `scp` binary being present on the target host.
- For large or frequent transfers, consider `rsync` over SSH: it supports delta
  transfers and is more efficient for repeated syncs.
- On RHEL 9 and newer systems, `scp` may use the SFTP subsystem by default.
  If transfers fail, check that `Subsystem sftp` is enabled in `sshd_config`.
