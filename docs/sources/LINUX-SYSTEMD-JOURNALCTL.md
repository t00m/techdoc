---
Category: Procedure
Command: journalctl, cp, du, systemctl
Date: 2024-05-14 12:00:00
DocType: Reference
Filename: journald.conf, journald.conf.back
Filesystem: /etc, /var, /var/log, /etc/systemd, /var/log/journal
OS: Linux
Priority: Normal
Product: Systemd
Scope: Unix Administration
Status: Released
Tag: clear, log, message, vacuum, unit, 
Team: Unix
Topic: Administration, Monitoring, Troubleshooting, Housekeeping
Website: linuxhandbook.com
---

# How to manage Systemd journal logs

## Introduction

The systemd journal is systemd's own logging system. It is equivalent to syslog in the init system. It collects and stores kernel logging data, system log messages, standard output, and errors for various systemd services.

The thing with logging is that over time, it starts to grow big. If you check the disk space in Linux, you'll see that sometimes it takes several GB of space.

## Check Logs

### By unit

To follow journal entries for a specific unit in real time:

```bash
sudo journalctl -f -u cups
```

where `-f` shows only recent journal entries and `-u` allows specifying the unit name (e.g. `cups`).

### By time

To query entries within a specific time window:

```bash
sudo journalctl --since "2024-10-11 00:30:00" --until "2024-10-11 03:00:00"
```

where `--since` and `--until` act as lower and upper time boundaries respectively.

## Clean Procedure

### Check disk space

First, check the space taken by journal logs with the `du` command:

```bash
du -sh /var/log/journal/
```

You can also use `journalctl` directly for the same purpose:

```bash
journalctl --disk-usage
```

### Rotate

The first recommended step is to rotate journal files. This marks currently active journal logs as archives and creates fresh log files. It is optional but considered good practice.

```bash
sudo journalctl --rotate
```

### Clear journal logs older than N days

Keep in mind that logs are important for auditing purposes, so avoid deleting all of them at once. To remove all entries older than two days:

```bash
sudo journalctl --vacuum-time=2d
```

### Restrict logs to a certain size

Another approach is to restrict the total log size. This deletes journal log files until the disk space occupied falls below the specified threshold.

```bash
sudo journalctl --vacuum-size=100M
```

### Restrict number of log files

The third method limits the number of archived log files. As logs grow older they are archived into separate files. To keep only five archive files:

```bash
journalctl --vacuum-files=5
```

### Automatically clearing old log files

The manual vacuum operations above clean the logs at the time of execution. In a month, logs will accumulate again. Rather than relying on manual intervention, systemd can be configured to handle old log files automatically.

The configuration file is located at `/etc/systemd/journald.conf`. Lines that are commented out indicate the compiled-in defaults (they are in effect even when commented).

The most relevant settings for automatic housekeeping are:

| Setting | Description |
| --- | --- |
| `SystemMaxUse` | Maximum disk space that journal logs may occupy |
| `SystemMaxFileSize` | Maximum size of an individual journal log file |
| `SystemMaxFiles` | Maximum number of journal log files to keep |

Make a backup of the configuration file before editing:

```bash
cp /etc/systemd/journald.conf /etc/systemd/journald.conf.back
```

Uncomment (remove the leading `#`) the setting you want to activate. The example below restricts the maximum journal disk space to 250 MB:

```ini
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.
#
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# Defaults can be restored by simply deleting this file.
#
# See journald.conf(5) for details.

[Journal]
#Storage=auto
#Compress=yes
#Seal=yes
#SplitMode=uid
#SyncIntervalSec=5m
#RateLimitIntervalSec=30s
#RateLimitBurst=1000
SystemMaxUse=250M
#SystemKeepFree=
#SystemMaxFileSize=
#SystemMaxFiles=100
```

After editing the configuration, restart the journal daemon to apply the changes:

```bash
sudo systemctl restart systemd-journald
```

> **NOTE:** `journald.conf` allows further tuning beyond housekeeping, including setting the minimum log level (e.g. `info`, `debug`, `err`). Refer to `man journald.conf` for the full parameter reference.

## References

* [Linux Handbook — Clear Systemd Journal Logs](https://linuxhandbook.com/clear-systemd-journal-logs/)
