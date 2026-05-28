---
Category: Procedure
Command: getent, nslookup
Date: 2021-01-22
DocType: Reference
Filename: /etc/hosts, /etc/nsswitch.conf
Filesystem: /etc
OS: Solaris
Priority: Normal
Scope: Unix Administration
Tag: network, DNS, hosts, nsswitch, resolve
Topic: Migration, Network
---

# Check IP or hostname resolution

## Information

How to check properly IP/hostname resolution

## Procedure

* `nslookup` will perform a DNS request
* `getent hosts <hostname>` will perform a resolution by looking first in `/etc/hosts`, and if not found, then in DNS (`/etc/nsswitch.conf`)
