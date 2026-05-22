---
Author: Tomas Virseda
Category: Procedure
Command: crontab, git, kb4it, nice, nohup, python3, ssh-add, ssh-agent, eval
Date: 2026-05-22
DocType: Tutorial
Filesystem: /home, /proc, /tmp, /var/log
OS: Linux
Product: GitLab, Markdown, KB4IT, Crontab
Protocol: SSH
Scope: Knowledge Management
Tag: automation, daemon, agent, repository, scripting
Topic: Documentation, Monitoring, Operation
---

# Keeping a KB4IT Website Up-to-Date Automatically

## Overview

This tutorial explains how to set up an automated pipeline that keeps a KB4IT-generated
website current without manual intervention. The KB4IT document sources are Markdown files
hosted in a GitLab repository. The pipeline clones or pulls from that repository before
each build so the published site always reflects the latest committed content.

The solution relies on four components:

1. **A shell launcher script** - the main entry point invoked by the scheduler.
2. **A process-check script** - ensures the file-notification daemon is always running.
3. **A repository-update script** - pulls the latest documents from GitLab before rebuilding.
4. **A cron job** - triggers the pipeline at a fixed interval.

By the end of this tutorial you will have a fully automated build loop: every few minutes
cron wakes up, verifies the notification daemon is alive (restarting it if needed), pulls
the latest Markdown documents from GitLab, and rebuilds the website with KB4IT.

## Prerequisites

- KB4IT installed and a `repo.json` configuration ready.
- A dedicated OS user (e.g. `kbuser`) whose `~/.bashrc` exports the required environment variables.
- Python 3 available at `/usr/bin/env python3`.
- A GitLab repository containing the KB4IT Markdown source documents.
- A GitLab personal access token with at least `read_repository` scope.
- SSH access to the server running the pipeline (key-based authentication recommended).
- Basic familiarity with cron, shell scripting, and Python.

---

## Step 1 - Create the Shell Launcher Script

The launcher is the single script that cron calls. It runs each component in order and
invokes KB4IT at reduced CPU priority to avoid impacting other workloads on the host.

Create the file `~/bin/kb4it_builder.sh` and make it executable (`chmod +x`):

```bash
#!/usr/bin/env bash
# kb4it_builder.sh - Main build launcher for the KB4IT pipeline

# 1. Ensure the file-notification daemon is running
/usr/bin/env python3 ~/bin/kb4it_check_notify.py

# 2. Pull the latest documents from the GitLab repository
/usr/bin/env python3 ~/bin/kb4it_update_repository.py

# 3. Rebuild the website at reduced CPU priority
nice -n 15 ~/.local/bin/kb4it build ~/kb4it/config/repo.json
```

> **NOTE:** The `nice -n 15` prefix tells the kernel to schedule the KB4IT build with
> below-normal CPU priority (value 15 out of a maximum of 19). This keeps the server
> responsive during builds.

---

## Step 2 - Create the Process-Check Script

This Python script scans `/proc` to determine whether the file-notification daemon
(`kb4it_notify.py`) is already running. If it is not, the script starts it in the
background using `nohup` and also starts an SSH agent so the daemon can access
key-protected remote repositories.

Create the file `~/bin/kb4it_check_notify.py` and make it executable (`chmod +x`):

```python
#!/usr/bin/env python3
"""
kb4it_check_notify.py
Checks whether the kb4it_notify.py daemon is running.
If not, it starts the daemon and an SSH agent.
"""

import os

# Scan /proc to detect whether the daemon is already running
notify_running = any(
    b'kb4it_notify.py' in open(os.path.join('/proc', pid, 'cmdline'), 'rb').read()
    for pid in os.listdir('/proc') if pid.isdigit()
)

if notify_running:
    print("Notification daemon already running - nothing to do")
else:
    os.system("nohup ~/bin/kb4it_notify.py &")
    print("Notification daemon started")
    os.system('eval "$(ssh-agent -s)"')
    print("SSH agent started")
```

> **NOTE:** Scanning `/proc` is a lightweight, dependency-free way to check for a running
> process on Linux. No external libraries are required.

---

## Step 3 - Create the Repository-Update Script

This script pulls the latest Markdown document sources from the GitLab repository via SSH,
using an existing SSH agent session discovered at runtime under `/tmp/ssh-*/*`.
It resolves the agent socket automatically by matching the socket owner to the
current OS user, then runs `git pull` with OAuth2 token authentication against GitLab.

> **NOTE:** Before SSH agent auto-discovery works, make sure the agent is running and
> your key is loaded:
> ```bash
> eval "$(ssh-agent -s)"
> ssh-add ~/.ssh/id_rsa
> ```
> The process-check script (Step 2) handles the agent startup for subsequent runs.

Create the file `~/bin/kb4it_update_repository.py` and make it executable (`chmod +x`):

```python
#!/usr/bin/env python3
"""
kb4it_update_repository.py
Discovers the running SSH agent for the current user, then pulls
the latest documents from the GitLab repository.
"""

import os
import pwd
import glob

# Git pull command template - SSH_AUTH_SOCK is injected at runtime
# Replace the GitLab URL and repository path with your own values
UPDATE_REPO  =  "export SSH_AUTH_SOCK=%s; "
UPDATE_REPO += "cd ~/kb4it/documents; "
UPDATE_REPO += "git pull https://oauth2:<ACCESS_TOKEN>@gitlab.example.com/<user>/kb4it-documents.git "
UPDATE_REPO += ">> /dev/null 2>&1"

# Discover SSH agent sockets in /tmp and match to the current user
current_user = os.environ['USER']
agent_socket = next(
    (p for p in glob.glob('/tmp/ssh-*/*')
     if pwd.getpwuid(os.stat(p).st_uid)[0] == current_user),
    None
)

# Run git pull only when a valid agent socket is found
if agent_socket and os.path.exists(agent_socket):
    print("Found SSH agent socket: %s" % agent_socket)
    os.system(UPDATE_REPO % agent_socket)
    print("GitLab repository updated")
else:
    print("ERROR: SSH agent not found - repository update skipped")
```

> **NOTE:** Replace `<ACCESS_TOKEN>` with a GitLab personal access token (or equivalent
> for your Git hosting provider) and adjust the repository URL and local path
> (`~/kb4it/documents`) to match your setup. Never commit a real token to version control;
> consider reading it from an environment variable or a credentials file with restricted
> permissions (`chmod 600`).

---

## Step 4 - Create the Log Directory

Ensure the log directory exists so output from the pipeline can be captured:

```bash
mkdir -p ~/logs
```

---

## Step 5 - Schedule the Pipeline with Cron

Open the crontab for the dedicated user:

```bash
crontab -e
```

Add the following line to run the pipeline every five minutes:

```
*/5 * * * * source ~/.bashrc && ~/bin/kb4it_builder.sh
```

The `source ~/.bashrc` call is important: it loads environment variables (Python paths,
KB4IT configuration, etc.) that are not available in the bare cron environment.

Verify the entry was saved:

```bash
crontab -l
```

---

## Step 6 - Verify the Setup

After saving the crontab, wait a few minutes and then check the output from the scripts.
Since output goes to stdout/stderr, redirect it from the cron entry if you want a
persistent log:

```
*/5 * * * * source ~/.bashrc && ~/bin/kb4it_builder.sh >> ~/logs/kb4it_build.log 2>&1
```

Then tail the file to confirm all three steps are running correctly:

```bash
tail -f ~/logs/kb4it_build.log
```

Expected output:

```
Notification daemon already running - nothing to do
Found SSH agent socket: /tmp/ssh-XXXXXXXXXX/agent.XXXX
GitLab repository updated
```

If the notification daemon was not running at the first execution, you will see:

```
Notification daemon started
SSH agent started
Found SSH agent socket: /tmp/ssh-XXXXXXXXXX/agent.XXXX
GitLab repository updated
```

---

## Summary

You now have a four-part automated pipeline:

| Component | File | Role |
|---|---|---|
| Launcher | `~/bin/kb4it_builder.sh` | Orchestrates the pipeline; called by cron |
| Process check | `~/bin/kb4it_check_notify.py` | Keeps the notification daemon alive |
| Repo update | `~/bin/kb4it_update_repository.py` | Pulls latest documents from GitLab |
| Scheduler | `crontab` entry | Triggers the launcher every 5 minutes |

Every five minutes, cron wakes up, the process-check script ensures the notification
daemon is running (starting it if necessary), the latest Markdown documents are pulled
from GitLab, and KB4IT rebuilds the site. The build runs at reduced CPU priority so
the host remains responsive throughout.

Adjust the cron interval and the GitLab repository URL to match your own infrastructure
and release cadence.
