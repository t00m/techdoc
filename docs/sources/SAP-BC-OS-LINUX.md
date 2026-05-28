---
Author: Tomás Vírseda
Bookmark: Yes
Category: Note
Command: nc, rsync, xsltproc, mailx, openssl, curl, find, dd, xargs, ls
Date: 2019-10-08
DocType: Reference
Filesystem: /oracle, /usr/sap
OS: Linux
Product: SUM, SSH
Scope: Unix Administration, SAP Basis
Status: Draft
Tag: useful, commands, basis
Topic: Operation, Troubleshooting
---

# Linux for Basis Admins

## Introduction

Useful commands for Basis Administrators working with Linux.

## Send email

To send the output of a program by email, use a pipe:

`| mailx -s "$(hostname) control_check_ora.log" adminsap@company.example`

## Find files

Find files modified in the last day and format the timestamp:

```bash
find /path/to/directory/* -mtime -1 | xargs ls -ldtrh --time-style="+%Y-%m-%d %H:%M:%S"

find /oracle/*/sapdata? | xargs ls -ldtrh --time-style="+%Y-%m-%d %H:%M:%S" | grep sr3.data
```

## Check ports

`nc -zv webserver.local 8080`

**Output:**

```
Connection to webserver.local 8080 port [tcp/*] succeeded!
```

## Create a large file

Create a 10G file:

```bash
dd if=/dev/zero of=/oracle/<SID>/DELETE_IF_NECESSARY.SAP bs=1 count=0 seek=10737418240
```

## Sync with remote directories using an SSH key

Sync remote directories using `rsync` over SSH instead of SFTP:

```bash
rsync -auvz -e "ssh -l root -i <SSH_PRIV_KEY>" <source_server>:/path/to/source/directory /local/target/path
```

## Generate HTML report after upgrade

Get all `.xml` and `.xsl` files from `/usr/sap/<SID>/SUM/abap/doc` (root directory only).

From a Linux box, run the following commands to generate the HTML reports:

```bash
xsltproc UpgAnalysis.xsl UPGANA.XML > report.html
xsltproc SAPupPhaseList.xsl phaselist.xml > phaselist.html
```

## Check or download certificates

Check certificate details for a web server:

```bash
curl -I https://webserver.company.example -v
```

Inspect the full certificate chain:

```bash
openssl s_client -showcerts -connect api.company.example:443
```

!!! note
    The commands above connect to the target host on port 443 and display the certificate chain, issuer, validity dates, and TLS session details.
